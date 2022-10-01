# Adjusting thermodynamic vertical profiles of reanalysis data

### Authors:
- Lead author: Ehsan Erfani @UW
- Collaborators: MCB group members @UW 

### Requirements:
- Python3 in Jupyter Notebook (installed by Anaconda).
- See the beginning of each code for the required Python3 libraries.

### The analyses are featured in:
Erfani, E., R. Wood, P. Blossey, S. Doherty, R. Eastman, 2022: Using a phase space of environmental variables to drive an ensemble of cloud-resolving simulations of low marine clouds, American Geophysical Union (AGU) Fall Meeting, Chiacago, IL, 12-16 Dec. 2022.

### Description:
This code utilizes satellite microwave observations of liquid water path (LWP) to modify reanalysis (in particular ERA5) temperature and moisture profiles. 
This is done by sharpening the profiles near the inversion level and is necessary for nudging the Large Eddy Simulation (LES) experiments. 
Without sharpening, the LES might not simulate the Stratocumulus cloud layer. This is done by finding the optimized profiles with the LWP equal to observed values.

### Inputs:
- Vertical profiles of T, qt, P, LWC, and RH from reanalysis.
- Microwave LWP.
- Some parameters can be changed to modify the profiles based on various conditions. Search for the word "parameter" in the code. Some important parameters are: inversion height (Zinv), T @Zinv, qt @Zinv, delta_T @Zinv, delta_qt @Zinv, ...



# 
### Science and Algorithm:


- Inputs: Tl_ctrl = T_ctrl + g*z/Cp - (L/Cp)*qc, qt_era (= qv+qc), Microwave LWP, ERA Zinv. Note that Tl is the liquid-water static energy divided by Cp, and
          qt is the total water mixing ratio.
          Note: ctrl refers to the initial ERA5 profile.
- Choose inversion height, zi: compute inversion height based on ERA ctrl profiles using min of d(RH)/dz*d(THETAL)/dz. 
                                Alternatively, use MODIS CTH.
- Build Tl(z) and qt(z) in two parts: FT and BL
- First FT: 

   o Let L_FT be some height above the inversion where ERA ctrl profiles don't "feel" the BL anymore, say 500m. 
   
   o For z>zi+L_FT:
   
   ![image](https://user-images.githubusercontent.com/28571068/114354208-c1f22f80-9b22-11eb-80be-0f83fd212239.png)
   
   o For zi < z < zi+L_FT, fit a line to the ctrl Tl/qt profiles away from the inversion and extrapolate down to the inversion. 
     This would be: Tl[zind2] = np.polyval(np.polyfit(z[zind], Tl_ctrl[zind], 1), z[zind2]) where zind are the indices where zi+L_FT < z < zi+3*L_FT 
     and zind2 has zi<z<zi+L_FT.  
   
   o Note: You might need to blend smoothly across zi+L_FT to avoid discontinuities.  
   
   o The top of the region where you're fitting the line is also arbitrary.  I chose  zi+3*L_FT as a reasonable first guess.

   o To ensure moisture profile sharpening above Zinv:
   
   ![image](https://user-images.githubusercontent.com/28571068/114354720-755b2400-9b23-11eb-9e33-8bf7f2b58b9c.png),  and   
   ![image](https://user-images.githubusercontent.com/28571068/114359502-e51fdd80-9b28-11eb-8b77-2c3397b62526.png)

    where delta_qt_inv is the difference between FT qt and BL qt near the inversion level.    


- Within the BL

  o The whole temperature and moisture profiles within the BL are:
  
  ![image](https://user-images.githubusercontent.com/28571068/114353870-5f992f00-9b22-11eb-971e-3b4117bb60bd.png),  and   
  ![image](https://user-images.githubusercontent.com/28571068/114354023-8e170a00-9b22-11eb-8887-9a104dd91545.png)
  

- How to solve:

  o takes inputs: Tl_ctrl(z), qt_ctrl(z), qt_inv, Tl_inv and ERA Zinv, and

  o follows the above scheme to produce outputs adj Tl(z) and ajd qt(z). (adj: adjusted).  

  o Make a second function that:

     - takes the inputs: Tl_ctrl(z), qt_ctrl(z), qt_inv, Tl_inv and Zinv along with LWP_target.
     - calls the first function to compute adj Tl(z) and adj qt(z)
     - computes the LWP of the resulting profile by computing LWC using saturation adjustment at each height, and
     - outputs a (positive) number that tells how well the resulting adj profile matches LWP_target while preserving the vertical integrals of the ERA ctrl Tl and qt profiles.

     ![image](https://user-images.githubusercontent.com/28571068/114470180-54d2ae80-9ba3-11eb-973c-e9bf943794a3.png)

     where Trho is density temperature:

     ![image](https://user-images.githubusercontent.com/28571068/114353678-2660bf00-9b22-11eb-8ac2-09bd2a062c7a.png)
  
     and:

     ![image](https://user-images.githubusercontent.com/28571068/114360809-4ac09980-9b2a-11eb-80e7-68c0175d7ad6.png)

     F is equal to 0.3

  o Automate the process of determining qt_inv and Tl_inv by using a function like "fmin" that will vary those inputs and choose the values that minimize the output function. 



# 
### Update, 2020-04-12:

- Make sure that in the FT, d(qt) / d(z) is always negative or zero right above the inversion level.
- Remove the equation containing the parameter delta_Tl_inv for the FT profile to avoid a non-physical Tl lapse rate of zero.
- In addition to Tl_inv, qt_inv, and Zinv, two more variables are added to the minimization function: delta_qt_inv and F. The function changes all these variables in order to find the minimum value of variable "A". This ensures minimum arbitrary assumptions.



# 
### Update, 2021-04-06:

- During the nudging period, qt and T profiles are constant, so it might make sense to make the winds (u and v) constant in time as well, but we would want to avoid having a huge transient when the geostrophic winds start to change at the end of the nudging period. 
- Since the winds do seem to change a lot through the nudging period, we find constant-in-time geostrophic wind profiles (ug and vg) that give wind speeds close to that specified in the soundings without needing to increase the strength of wind nudging. This is done explicitly above the boundary layer by including forcing due to the effect of large-scale subsidence (W_LS). In the MBL, we use a uniform value down to the surface. This will prevent abrupt jumps in ug and vg when the nudging ends.

![image](https://user-images.githubusercontent.com/28571068/162072925-216a373f-68f8-41ad-bc0d-b5751ce7001a.png)

- Sections are added to save the qt, T, u, v, ug, vg profiles to the forcing file.
