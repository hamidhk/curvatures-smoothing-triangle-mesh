# Three Phase Triangular Mesh Extraction and Smoothing

 
This program is developed in order to extract and smooth triangulated mesh surfaces for 3D images containing 3 phases. Smoothing can be performed by either isotropic or anisotropic diffusion. The latter has heavier computation where Gaussian and principal curvatures are calculated in addition to the mean curvatures. The meshes are smoothed using cotangent discretization of the Laplace-Beltrami operator for manifolds. For details of the smoothing method, check

["Discrete Differential-Geometry Operators for Triangulated 2-Manifolds"](http://www.geometry.caltech.edu/pubs/DMSB_III.pdf)
by Meyer, Desbrun, Schroderl, Barr, 2003, Springer
"Visualization and Mathematics III" book, pp 35-57.

In case of fluid flow in porous media, after smoothing fluid–fluid and solid–fluid meshes, the mean curvatures at each interface is calculated. This can be used to find the local capillary pressure in 2-phase flow in porous material. The contact angles between fluid–fluid and solid–fluid meshes on the 3-phase contact line can also be calculated.

Further, the code can be used in general for smoothing any triangular meshes, finding intersection of two-three arbitrary meshes, correcting non-orientable vertices in 2D and 3D intersections, creating neighborhood maps and data structure required for curvature calculation, detection of mesh boundaries etc.

Computing curvatures and mesh smoothing is inherently a heavy iterative computation. I suggest you slice a smaller volume of your image in main() after reading the image for testing, before you run the code on whole image. Make sure the sliced volume contains all 3 phases. A 100x100x100 sample image was uploaded in the 'sample_input' folder; and 'sample_output' is the output for the same image. Four screen shots of original and smoothed meshes were also uploaded in the main project folder.

This is program is used to find contact angles and curvatures of X-ray tomography images of water–oil saturated porous solid samples. The outcome is summarized in a peer-reviewed article. I will upload the article or provide a download link here, as soon as the review process is over. 

## Descriptions

### 1)
The .raw/.mhd images used as example image can be loaded as numpy arrays either using simpleITK library, or simply by simpleitk plugin of skimage library after simpleitk library is installed. If your image is in a  different format, change the code accordingly in the main() function.
The image should be converted to a 3D numpy array with the same shape (dimensions) as the input image.

The input image has to contain 3 phases, and it has to be a segmented image. Global variables 'Aval', 'Bval' and 'Sval' early in the code are the values of phase A (wetting fluid), phase B (nonwetting fluid) and solid. Adjust them according to the values in your segmented image. In the images used in this work, pixel value of 0 is nonwetting fluid (Dodecane), 1 is wetting fluid (brine) and 2 is solid (sintered glass spheres) plus the boundary.

The other global variable 'path' is simply the address of folder where the image is stored on the disk. The program will save the outputs in the same folder with image name as prefix.

### 2)
Smoothing of fluid–fluid interfaces should be done locally. This requires identification of individual interfaces. Therefore, the wetting and nonwetting droplets are labeled separately using ndimage.measurements.label(img_phaseA) and ndimage.measurements.label(img_phaseB) from scipy library. This is done in the beginning of function,

```
ThreePhaseIntersection(img, **kwargs)
```
Intersection function receives the segmented 3-phase image 'img' and the values of each phase are already given as global variables Aval, Bval, Sval.

intersectAlongAllAxes(args), a nested function in the intersection function extracts the intersections between the phases and all the necessary information for creation of the meshes and corrections of non-orientable vertices performed later.

ThreePhaseIntersection is a thorough function. In addition to finding intersections, it performs labeling corrections, mesh orientation corrections, finding unique indexes for vertices (points), labeling the vertices to identify which interface they belong to, and construction of 15 arrays/structures in vector form required for finding normal vectors, curvatures and mesh smoothing etc.

Function runtime was roughly 673 seconds for an image with dimensions of (576, 751, 751), with about 28.5 million vertices, 3.7 million triangles at AB (fluid–fluid), and 53.4 million triangles at S-AB (solid–fluid) meshes. The runtime for the same image where only meshes were extracted/corrected and the smoothing structures were ignored was 355 seconds.

If kwarg return_smoothing_structures=True is passed as an argument to the function, it will return all the structures required for vectorized computations. Otherwise only necessary arrays are returned. The necessary arrays are enough to construct/save mesh in .stl format, and to recognize which pairs of phases - phase A (wetting), phase B (nonwetting) or phase S (solid) - a vertex belongs to.
```
ThreePhaseIntersection(img, return_smoothing_structures=True)
```

Further, the Intersection function provides thorough information on the individual fluid clusters which are not in contact with image boundaries. Information include topology (Euler characteristic), vertices on the cluster, triangle faces of solid–fluid belonging to cluster, triangle faces of fluid–fluid belonging to cluster. 3-phase contact lines of the cluster are also provided as a time series in which consecutive vertices are sorted. This can be used later after smoothing to study the local variations of contact angles. The function
```
def clusterBasedContactCalculations(verts, verts_, facesi, nVi, ang, kGi, kGs, Avori, Avors, labc, cluster, path_, filename):
```
performs cluster-based calculations after smoothing. verts and verts_ are original and smoothed vertices, respectively.


Below, description of important structures follows.

1) verts: float64 numpy array with the 3D (z,y,x) coordinates of the vertices!
2) facess: int64 numpy array of triangles in the solid 's' mesh (wetting-solid and nonwetting-solid intersections all together). Every triangle composed of 3 indexes from verts array.
3) facesi: the same as above, for interface 'i' mesh (intersections of wetting (phase A) & nonwetting (phase B))
- - throughout the code 's' and 'i' come at the end of structures/arrays for solid–fluid and fluid–fluid meshes, respectively - -
4) labc: int64 numpy array with the same length as verts! labc has 3 columns. It gives the labels for the vertices. If a vertex is in the AB interface, first column is label from phase A (fluid A) and 2nd column is label from phase B (fluid B). If a vertex is at S-AB (solid–fluid) mesh, first column is label from solid and 2nd column is label from either A or B. The integers -1, -2, -3 and -4 come at the third column, where -3 and -2 mean the point comes from phaseA-solid and phaseB-solid intersections, respectively. Value -1 means the point comes from phaseA-phaseB (fluid–fluid) intersection; and the value -4 shows the point is on the 3-phase contact line (common between 3 phases). When third column, labc[:,2], is -4, the labels in 1st and 2nd column are taken from phase A and phase B. This is convenient because fluid–fluid labels are needed later for recognition of individual interfaces.
5) interface: is a list based on labc, and constructed only for AB intersections (not for solid). Every element in list has 5 sub-elements: [ label from phase A, label from phase B, indices of verts @ this interface, indices of facesi @ this interface ]. In principle, this structure defines individual interfaces. Having the 'interface', calculation of mean curvature etc will be simple and fast for individual fluid–fluid interfaces.

### 3)
Smoothing requires the calculation of normal vectors at faces (triangles) and verts (vertices); and it also requires the normalization of vectors (length = 1). Small and effective functions for these purposes are created using numpy library.

``` 
facesUnitNormals(verts, faces) 	  					# normals of faces
verticesUnitNormals(verts, faces, nbrfc, msk_nbrfc) # normals of verts
unitVector(vec)					 					# normalization of vectors (length = 1)
```

### 4)
The function

```
smoothingThreePhase(verts,facesi, facess, nbrfci, nbrfcs, msk_nbrfci, msk_nbrfcs, nbri, nbrs, ind_nbri, ind_nbrs, msk_nbri, msk_nbrs, interface, **kwargs)

```

receives verts, facesi, facess etc. It changes the positions of verts along their unit normal vectors to maximize their dot product on the solid–fluid mesh, and to minimize sum of standard deviation of the mean curvatures on the fluid–fluid mesh.

The amount of change is determined by the weight function (mean curvature) in isotropic smoothing, and slightly complicated in the anisotropic smoothing. The default smoothing is isotropic which is also the goal of this project.

Vertices in solid–fluid and fluid–fluid meshes are updated simultaneously at each iteration. In this way, the three-phase contact line becomes automatically smooth as well.

t_f and t_s are tuning parameters. They are initially set to t_s=0.15 and t_f=0.3. The reason to use fractions is to avoid too large changes which can cause divergence. This is necessary in complex structures like porous media images.

The variables tau_s and tau_f are defined as goals of smoothing.
```
tau_s = 1 - mean_of_dot_product_of_all_neighbor_vertices_on_solid_fluid_mesh
tau_f = mean_of_standard_deviation_of_mean_curvatures_on_fluid_fluid_mesh
```
Both tau_s and tau_f have to be minimized to smooth the meshes. t and tau variables are used to run and stop the iteration. For instance, when tau_s improves in an iteration, t_s will be multiplied by 1.05,  and if it gets worse t_s will reduce to 0.8 of its previous value. t_f is updated in the same manner. When one of t variables go below 1e-3, which happens in the later steps of smoothing when there is nearly no changes, both t_s and t_f are set to minimum of these two and a final count_down for iterations starts. The loop kills the iteration after 5 more steps. The step which provided the least standard deviation for mean curvatures of the fluid–fluid interfaces is picked as the result of iteration. 

Smoothing function updates the vertices at every iteration with two constraints. First constraint ensures that the vertices do not displace more than 0.2 pixel length at each step. If displacement is more than this, it will be reduced to 0.2. Second constraint ensures that the vertices never go farther than 1.7 pixel length from their original positions. 1.7 is roughly the longest diameter of 1x1x1 voxel. Freedom of displacement of 1.7 is associated with the uncertainty in the experimental images. The meshes smooth often long before the vertices have displaced by 1.7 pixel. The user can introduce a different constraint by using verts_constraint kwarg as an input for the smoothing function. Tests show that the resulted smooth meshes do not vary much with values roughly in the range of 1-1.7. Only minor number of vertices may need a large displacement.
```
# verts_constraint=1.4
new_verts = smoothingThreePhase(args, verts_constraint=1.4)
```

Smoothing and calculation of curvature/weights are done by isotropic diffusion by default. If anisotropic method is required, keyword argument should be passed to the smoothing function. Smoothing function automatically passes the argument to the curvature function.
```
# method='aniso_diff' 
new_verts = moothingThreePhase(args, method='aniso_diff')
```
Smoothing of a full image with specifications mentioned in section 2, stopped after 69 iteration steps. Each iteration took roughly 173 seconds.

### 5)
After smoothing is performed, based on the smoothed vertices, main() function computes normal vectors on solid–fluid (nVs) and on fluid–fluid (nVi) meshes, in addition to the mean curvatures on fluid–fluid (kHi), average mean curvature for individual fluid–fluid interfaces (khm), interfacial areas (Ai), and finally contact angles (ang) on the three phase common line. Note that all the arrays nVs, nVi, ang, kHi and ang have the same length as verts (vertices). This ensures that the indexes in these arrays are the same as indexes for verts. For instance, nVi is (0,0,0) wherever the index is for a vertex on the solid-solid mesh away from the 3-phase common line. All these arrays can be simply masked/sliced by the label array (labc). For instance, binary mask 
```
labc[:,2]==-4 
```
returns all the indexes on the contact lines; or binary mask
```
numpy.logical_or(labc[:,2]==-1, labc[:,2]==-4)
```
returns all the indexes on the fluid–fluid interfaces; and the mask
```
numpy.logical_or(labc[:,2]==-2, labc[:,2]==-3)
```
returns all the indexes on the solid mesh excluding the ones on the 3-phase common lines.

All arrays of verts (initial vertices), labc, facesi, facess, verts_ (smooth), nVi (smooth), nVs (smooth), ang (smooth), kHi (smooth) are saved by numpy.save function with a prefix containing the image name, and _init_ or _final_. Arrays can later be loaded using numpy.load for further analysis.

Contact angles are also separately saved as a 'csv' file.

The list 'interface' introduced in section 2, is modified here to add info on integral of mean curvature, area and mean contact angle for individual interfaces. Any interface with less than 100 vertices is removed from the list (smaller interfaces are too uncertain in computations). The list elements will be 
```
[ 'labelPhaseA, labelPhaseB, meanCurv(1/pixel), Area(pixel**2), meanAngle(deg), indicesOfABVertices, indicesOfABFaces' ]
```
with the first element as the header. The list is saved by pickle.dump function, and can be read with pickle.load.
An array similar to 'interface' list is saved together with _final_ arrays. Elements in the array are the same as those in the 'interface' list, except the last two columns which give the number of vertices and faces instead of the indexes.

Based on the list 'cluster' the cluster-based calculations are performed by 
```
clusterBasedContactCalculations(verts, verts_, facesi, nVi, ang, kGi, kGs, Avori, Avors, labc, cluster, path_, filename)
```
as mentioned earlier. Similar to 'interface', the results of cluster-based calculations are saved as both list and array.

The main function also prints the results of interface-based and cluster-based calculations.

There are a number of code blocks in main() function with description on top of the block. These block create/save mesh with .stl format; visualize the smooth/original meshes, 3-phase contact points, normal vectors, etc; visualize mean curvature sign; and visualize individual fluid–fluid interfaces with different colors. Visualizations are performed using malb function of mayavi library. Histogram for contact angle frequency is also created/saved.

### NOTE: 
The code is in the file smoothing_3phase.py. A number of figures/output are also included.
The results after smoothing are saved. The script reload_and_analyze.py can be used to reload the saved results for further analysis, for instance further cluster-based investigations or visualization of clusters and related results.


```
#smoothing #triangular #mesh #Laplace #Beltrami
#differential # geometry #cotangent #discretization
#mean #Gaussian #principal #curvature
#torus #double_torus #triple_torus #sphere #ball
#manifold #visualization #neighborhood #map
#vertex #vertices #edge #face #normal #vector
#Young-Laplace #capillary #pressure 
#X-ray #tomography #image #fluid #flow #porous #material
#Helmholtz #free #energy #state #variable #line
#vectorized #computation #python #python3
#numpy #skimage #scipy #ndimage #matplotlib #mayavi 
```
