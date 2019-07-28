This is a supplemental vignette for Oyafuso et al. (In Review) . This method is also used in Oyafuso et al. (2019). Code and techniques from other workers are used in this script. Please refer to Beyer et al. (2016) for the linearization of the compactness term used in their paper to linearize the boundary length term in the Marxan objective function. Also see M. Strimas-Mackey's article entitled, "Field Guide to ILP Solvers in R for Conservation Prioritization," on his gitHub account for the setup of the solver.

### Import necessary libraries (install if needed)
```{r Import Libraries, message = F}
library(kableExtra)
library(Matrix)
library(slam)
library(sp)
library(raster)
library(glpkAPI)
library(RColorBrewer)
```

### Set working directory 
Set your working directory to include domain.RData
```{r Set wd here}
setwd("/Users/zackoyafuso/Google Drive/GitHub/ILP-Reserve_Selection_Model/")
```

### Load spatial domain
```{r import data}
load('domain.RData')
```

```{r domain characteristics, echo = F}
#Number of Columns
nc = domain@ncols
#Number of Rows
nr = domain@nrows
#Total number of raster cells
N = nr*nc
#Total number of planning units
n_pus = length(na.omit(values(domain)))
```

The raster `domain` has **`r nc`** columns and **`r nr`** rows. Of the **`r N`** raster cells, there are **`r n_pus`** cells to be included in the analysis. These cells are called planning units (PUs) and will be referenced as such hereafter.

### Data Layers
This is what the spatial domain looks like at 2 km resolution. The brown cells are the PUs and the unfilled cells are cells in the domain of the raster but not included in the analysis. The two other major data layers are the opportunity cost and species (or conservation) layers.
```{r plot data layers, echo = F}
##Plot Spatial Domain
full_domain = domain
values(full_domain)[is.na(values(full_domain))] = 1
par(mar = c(0,0,0,0))
plot(raster::rasterToPolygons(full_domain),
     border = 'grey')
plot(raster::rasterToPolygons(domain), 
     axes = F,
     col = 'chocolate4',
     add = T)
text(x = mean(domain@extent[1:2]),
     y = mean(domain@extent[3:4]),
     paste('N =', n_pus, '\nplanning units'), cex = 2)
## Plot Opportunity Cost Layer
full_domain = domain
values(full_domain)[is.na(values(full_domain))] = 1
par(mar = c(0,0,0,0))
plot(mean_effort, axes = F, col = brewer.pal(9, 'Greens'))
plot(raster::rasterToPolygons(domain), add = T, border = 'darkgrey')
text(x = mean(domain@extent[1:2]),
     y = mean(domain@extent[3:4]),
     "Opportunity\nCost Layer", cex = 2)
## Plot Conservation Feature Layer
plot(spp_2km, axes = F, col = brewer.pal(9, 'Blues'))
plot(raster::rasterToPolygons(domain), add = T, border = 'darkgrey')
text(x = mean(domain@extent[1:2]),
     y = mean(domain@extent[3:4]),
     "Conservation\nFeature Layer", cex = 2)
```

### Spatial Compactness Objective
The compactness objective is defined as the total number of adjacent chosen PUs.  This entails calculating the cardinal PU neighbors for each PU. One way to do this is to calculate whether a planning unit is a corner, edge, or inner cell on the raster domain. Depending on the type of cell, its neighbors can be correctly calculated. The cells are indexed in a horizontal reading frame. Then for each PU, we calcualte the adjacent neighboring planning units. This information is stored in a matrix called `neighbor_matrix` with `r n_pus` rows and 4 columns (for each cardinal direction). 

```{r Cell Type, echo = F }
#Corner Indices
nw_corner = 1 #northwest corner
ne_corner = nc #northeast corner
sw_corner = N - nc + 1 #southwest corner
se_corner = N #southeast corner
#Edge (non-corner) indices
n_edge = (1:nc)
n_edge = n_edge[!(n_edge %in% c(nw_corner,ne_corner))] #remove corners
s_edge = ((N-nc):N)
s_edge = s_edge[!(s_edge %in% c(sw_corner, se_corner))] #remove corners
w_edge = (1:nr)*nc - nc + 1
w_edge = w_edge[!(w_edge %in% c(sw_corner, nw_corner))] #remove corners
e_edge = ((1:nr)*nc)
e_edge = e_edge[!(e_edge %in% c(se_corner, ne_corner))] #remove corners
inner_cells = (1:N)[-c(nw_corner, ne_corner, sw_corner, se_corner,
                       n_edge, s_edge, w_edge, e_edge)] 
```

```{r cell neighbors, echo = F}
#Planning Unit Indexes (horizontal reading frame)
pu_idx_hori = which(!is.na(values(domain)) == T)
###################################
## Neighbor Matrix
## We have to specify the indices of the neighboring cells for each cell
###################################
neighbor_matrix = matrix(data = NA,
                         nrow = n_pus, 
                         ncol = 4, 
                         dimnames = list(pu_idx_hori, c("W","E","N","S")))
for(idx in pu_idx_hori){
  
  row_idx = paste(idx)
  cell_type = sapply(X = list(nw_corner, ne_corner, sw_corner, se_corner,
                              n_edge, s_edge, w_edge, e_edge, inner_cells), 
                     FUN = function(x) any(idx == x))
  cell_type = which(cell_type == T)
  
  if(cell_type == 1) { #nw corner, record the E and S neighbors
    neighbor_matrix[row_idx, c('E', 'S')] = c(nw_corner + 1, nw_corner + nc)
  }
  
  if(cell_type == 2) { #ne corner, record the W and S neighbors
    neighbor_matrix[row_idx, c('W', 'S')] = c(nc - 1, nc*2)
  }
  
  if(cell_type == 3) { #sw corner, record the E and N neighbors
    neighbor_matrix[row_idx, c('E', 'N')] = c(sw_corner + 1, sw_corner-nc)
  }
  
  if(cell_type == 4) { #se corner, record the W and N neighbors
    neighbor_matrix[row_idx, c('W', 'N')] = c(se_corner - 1, se_corner-nc)
  }
  
  if(cell_type == 5) { #N edge, record the W, E, and S neighbors
    neighbor_matrix[row_idx, c('W','E','N')] = c(idx - 1, idx + 1, idx + nc)
  }
  
  if(cell_type == 6) { #S edge, record the W, E, and N neighbors
    neighbor_matrix[row_idx, c('W','E','N')] = c(idx - 1, idx + 1, idx - nc)
  }
  
  if(cell_type == 7) { #W edge, record the E, N, and S neighbors
    neighbor_matrix[row_idx, c('E','N','S')] = c(idx + 1, idx - nc, idx + nc)
  }
  
  if(cell_type == 8) { #E edge, record the W, N, and S neighbors
    neighbor_matrix[row_idx, c('W','N','S')] = c(idx - 1, idx - nc, idx + nc)
  }
  
  if(cell_type == 9) { #innr, record the N, S, E, and W neighbors
    neighbor_matrix[row_idx, c('N','S','W','E')] = c(idx - nc, idx + nc,
                                                     idx - 1, idx + 1)
  }
}
```

Because most of the cells in the raster domain are not included in the analysis, we now change the indexing system to only include the cells that are planning units. 
```{r echo = F}
###################################
## Convert the index of a cell to a cleaner index system
################################### 
new_idx = 1:n_pus
names(new_idx) = pu_idx_hori
#Remove neighbors that are not fellow planning units
neighbor_matrix[!(neighbor_matrix %in% pu_idx_hori)] = NA
#Translate the old indices to the new indices
neighbor_matrix[!is.na(neighbor_matrix)] = new_idx[paste(na.omit(as.vector(neighbor_matrix)))]
rownames(neighbor_matrix) = 1:n_pus
#Create dataframe of data information
df = data.frame(new_idx = 1:n_pus,
                old_idx = pu_idx_hori,
                cost = round(na.omit(values(mean_effort)), 2 ),
                spp = round(na.omit(values(spp_2km)), 3)  )
kable(head(neighbor_matrix), row.names = T) %>% kable_styling()
```

A preview of the data layer information is provided below:
```{r echo = F}
kable(head(df)) %>% kable_styling()
```

### Objective Functions
We define two objective functions first, opportunity cost and conservation value. 
$$ 
argmin 
\begin{equation}
\sum_{i=1}^{N} c_i x_i \text{ Opportunity Cost}
\end{equation}
$$

$$ 
argmax 
\begin{equation}
\sum_{i=1}^{N} r_i x_i \text{ Conservation Value}
\end{equation}
$$

$x_i$: binary decision variable, 1 if $i^{th}$ PU is chosen, 0 otherwise.

$c_i$: opportunity cost of the $i^{th}$ PU.

$r_i$: conservation value of the $i^{th}$ PU.

Total area is often as a target decided a priori. We specify this constraint using an inequality and maximum total area ($A$):

$$ s.t.
\begin{equation}
\sum_{i=1}^{N} x_i \leq A
\end{equation}
$$

At this point, there are **`r n_pus`** decision variables. The compactness objective is an inherently non-linear process, as it involves the interactions of PUs. A few steps are added to linearize the compactness objective. First, we introduce a new binary decision variable, $z_{ij}$ denoting the selection of adjacent cells $i$ and $j$. There are a total of $M$ cell adjacencies in the spatial domain while $E$ comprises the set of neighboring planning units. Thus, the compactness objective is defined as: 

$$ argmax
\begin{equation}
\sum_{(i,j)\in E} b_{ij}
\end{equation}
$$

We also need to add three constraints to make sure that if $z_{ij}$ is chosen (i.e., $z_{ij} = 1$), $x_i$ and $x_j$ are chosen as well (i.e., $x_i = x_j = 1$), and vice versa:

1) $z_{ij} - x_i \leq 0$
2) $z_{ij} - x_j \leq 0$
3) $z_{ij} - x_i -x_j \geq -1$

The constraints can be written in a linear (i.e., matrix) format, and the coefficients of the constraints can be formatted conveniently using a constraint matrix with the number of columns corresponding to the number of decision variables (N + M) and the number of rows corresponding to the number of constraints (3M + other constraints added in the model). This is generally a very large and sparse (i.e., lots of zeros) matrix. In R, sparse matrices are constructed by feeding the `Matrix::sparseMatrix()` function three vectors that correspond to the non-zero elements of the matrix: 

1) `mc` is a vector of column indices;
2) `mr` is a vector of row indices;
3) `mz` is a vector of coefficients;

With the sparse matrix are two vectors for the inequalities of the constraints (`sense`) and the value of the right hand side of the inequality (`rhs`)

```{r}
mc = mr = mz = sense = rhs = c()
## index for the vectors of the sparse constraints matrix
idx = 1
# index for the current row (cr)
cr = 1
# index for the zth added boundary decision variable
z = n_pus+1
M = 0 #cumulative number of cell interactions
for (i in 1:n_pus){ #for each true pu
  #which pus are neighboring?
  ids <- neighbor_matrix[i,which(neighbor_matrix[i,]>0)] 
  
  #To ensure we don't double count cell adjacencies
  if(all(ids < i)) next 
  
  unique_ids = ids[ids > i]
  
  #add to the number of interactions
  M = M + length(unique_ids)
  
  for (j in 1:length(unique_ids)){ #for each of the neighbors
    
    # First constraint: z_ij - x_i <= 0
    mc[idx] <- i       #ith pu
    mr[idx] <- cr      
    mz[idx] <- -1      #corresponds to -x_i
    idx <- idx + 1  
    mc[idx] <- z       #zth obj val (z = z-N)
    mr[idx] <- cr      
    mz[idx] <- 1       #corresponds to +z_ij
    sense[cr] = "<="
    rhs[cr] = 0
    idx <- idx + 1
    cr <- cr + 1       #Next row (constraint)
    
    # Second Constraint: z_ij - x_j <= 0
    mc[idx] <- unique_ids[j]  #jth neighboring pu
    mr[idx] <- cr
    mz[idx] <- -1      #corresponds to -x_j
    idx <- idx + 1
    mc[idx] <- z       #zth obj val (z = z-N)
    mr[idx] <- cr
    mz[idx] <- 1       #corresponds to z_ij   
    sense[cr] = "<="
    rhs[cr] = 0
    idx <- idx + 1
    cr <- cr + 1       #Next row (constraint)
    
    # Third Constraint: zij - xi - xj >= -1
    mc[idx] = i        #ith pu
    mr[idx] <- cr      
    mz[idx] <- -1      #corresponds to -x_i
    idx <- idx + 1  
    mc[idx] <- unique_ids[j]  #jth neighboring pu
    mr[idx] <- cr
    mz[idx] <- -1      #corresponds to -x_j
    idx <- idx + 1
    mc[idx] <- z       #zth obj val (z = z-N)
    mr[idx] <- cr
    mz[idx] <- 1       #corresponds to z_ij   
    idx <- idx + 1
    
    sense[cr] = '>='   #Switch equality to >=
    rhs[cr]   = -1     
    
    cr <- cr + 1       #Next row (constraint)
    z <- z + 1         #Next column (i.e., decison variable)
  }
}
cr = cr - 1
```

#### Double-Check

To double check, the constraint matrix at this point should have number of rows corresponding to thrice the number of cell-to-cell adjacencies (M)

```{r}
3*M == max(mr)
```
There should be n_pus + M columns (`r n_pus` planning units plus `r M` adjacencies)

```{r}
n_pus+M == max(mc)
```
# Solving the ILP model
```{r}
do_glpkAPI = function(
  area_constraint = 0.3,
  species_constraint = 0.2,
  boundary_constraint = 0.2,
  gap = 0.01,
  time_limit = Inf,
  bound = NA
){
  ################################
  ## Model Setup
  ################################
  # Opportunity Cost Vector. Because the length of the decision variable
  # vector is n_pus + M, the coefficients for the objective values consist of
  # the opportunity cost for each of the PUs, concatenated with a zero vector
  # of length M will not interact with the value of the objective function.
  objvals = c(df$cost, rep(0, M))
  
  # Add total area constraint to the constraint matrix
  cr = cr + 1 #Start a new row/constraint
  mr = c(mr, rep(cr, (z-1) ) )
  mc = c(mc, 1:(z-1) ) 
  mz = c(mz, 
         rep(1, n_pus) , #each PU has the same area
         rep(0, (z-1) - n_pus) #zeros for the z_ijs
  )
  
  # Add total conservation value constraint to the constraint matrix 
  cr = cr + 1 #Start a new row/constraint
  mr = c(mr, rep(cr, (z-1)))
  mc = c(mc, 1:(z-1) )
  mz = c(mz, df$spp, # conservation value of each PU
         rep(0, M)   # zeros for the z_ij's
  )
  
  # Add a total spatial compactness constraint to the constraint matrix 
  cr = cr + 1 #Start a new row/constraint
  mr = c(mr, rep(cr, (z-1)))
  mc = c(mc, 1:(z-1))
  mz = c(mz, rep(0, n_pus),  # zeros for the x_i's
         rep(1, M)           # ones for the z_ijs
  )
  
  # Create the sparse constraint matrix
  rij = sparseMatrix(i=mr, 
                     j=mc, 
                     x=mz)
  
  ##############################
  ## Defining the Model
  ###############################
  
  # Initialize Model
  model <- glpkAPI::initProbGLPK()
  
  # Set objective function as a minimization funtion
  glpkAPI::setObjDirGLPK(lp = model, 
                         lpdir = glpkAPI::GLP_MIN)
  
  # Initialize decision variables (columns)
  glpkAPI::addColsGLPK(lp = model, 
                       ncols = length(objvals) )
  
  # Set the objective function, specify no bounds on decision variables
  # GLP_FR means free variable
  glpkAPI::setColsBndsObjCoefsGLPK(lp = model, 
                                   j = seq_along(objvals),
                                   lb = NULL, ub = NULL,
                                   obj_coef = objvals,
                                   type = rep(glpkAPI::GLP_FR, length(objvals)))
  
  # Specify that decision variables are binary. GLP_BV means binary variable
  glpkAPI::setColsKindGLPK(lp = model, 
                           j = seq_along(objvals),
                           kind = rep(glpkAPI::GLP_BV, length(objvals)))
  
  # Initialize the structural constraints (rows)
  # There are 3*M constraints to set up the compactness objective
  # plus 3 constraints on total area, conservation, and compactness
  glpkAPI::addRowsGLPK(lp = model, 
                       nrows = 3*M + 3)
  
  # set non-zero elements of constraint matrix
  glpkAPI::loadMatrixGLPK(lp = model, 
                          ne = length(mz),
                          ia = mr, ja = mc, ra = mz)
  
  # Set the lower and upper bounds for the right-hand side
  # of the inequalities
  lower = c(rep(c(-Inf, -Inf, -1), times = M),
            0, #LB for area
            species_constraint*sum(df$spp), #LB for total cons. val.
            M*boundary_constraint) #LB for the total compactness
  
  upper = c(rep(c(0,0,Inf), times = M),
            area_constraint * n_pus, #UP for area
            Inf, #UB for total conservation value
            Inf) #UB for the total compactness
  
  # Specify the type of structural variable. Integers refer to different types.   # See ?glpkAPI::glpkConstants() for the set of variable types. 
  # "2" refers to a variable with a lower bound and 
  # "3" refers a variable with an upper bound
  bound_type = c(rep(c(3,3,2), times = M),
                 3,
                 2,
                 2)
  
  glpkAPI::setRowsBndsGLPK(lp = model, 
                           i = seq_along(upper),
                           lb = lower, ub = upper,
                           type = bound_type )
  
  # Presolve and automatically calculate relaxed solution
  # otherwise glpkAPI::solveSimplexGLPK(model) must be called first
  glpkAPI::setMIPParmGLPK(PRESOLVE, GLP_ON)
  glpkAPI::setMIPParmGLPK(MSG_LEV, GLP_MSG_ALL)
  
  # Gap to optimality
  glpkAPI::setMIPParmGLPK(MIP_GAP , gap)
  
  # Stop after specified number of seconds, convert to milliseconds
  if (is.finite(time_limit)) {
    glpkAPI::setMIPParmGLPK(TM_LIM, 1000 * time_limit)
  }
  
  # Solve Model
  t <- system.time({
    screen_out <- capture.output(glpkAPI::solveMIPGLPK(model))
  })
  
  #print(screen_out)
  
  # Extract lower bound from screen output
  if (is.na(bound)) {
    bound <- stringr::str_subset(screen_out, "^\\+")
    bound <- stringr::str_subset(bound, ">=\\s+(-?[+.e0-9]+)")
    bound <- stringr::str_match(bound, ">=\\s+(-?[+.e0-9]+)")
    bound <- ifelse(nrow(bound) > 0, as.numeric(bound[nrow(bound), 2]), NA)
  }
  
  # Prepare return object
  results <- list(time = summary(t)[["user"]],
                  output_code = screen_out,
                  x = glpkAPI::mipColsValGLPK(model),
                  objval = glpkAPI::mipObjValGLPK(model),
                  objbound = bound,
                  gap = (glpkAPI::mipObjValGLPK(model) / bound - 1))
  
  #Delete model
  glpkAPI::delProbGLPK(lp = model)
  
  return(results)
}
```

#### Example Run
```{r}
res_glpkAPI = do_glpkAPI(area_constraint = 0.3, 
                         species_constraint = 0.3,
                         boundary_constraint = 0.20,
                         gap = 0.01, 
                         time_limit = Inf, bound = NA)
```

### Result Check

The objective value outputted from the solver is `r res_glpkAPI$objval`. The total area constraint is 0.3, thus the solution should have a total area of at maximum 0.3.
```{r}
chosen_pus_glpkAPI = which(res_glpkAPI$x[1:n_pus] == 1)
length(chosen_pus_glpkAPI) / n_pus
```

The conservation value constraint is 0.3, thus the solution should have a conservation value value of at least 0.3.
```{r}
sum(df$spp[chosen_pus_glpkAPI]) / sum(df$spp)
```

The boundary constraint is 0.2, thus the solution should have a compactness value of at least 0.2.
```{r}
sum(res_glpkAPI$x[-1*c(1:n_pus)]) / M
```

### Plot solution (green)
```{r echo = F}
old_idx = as.numeric(names(new_idx)[chosen_pus_glpkAPI])
temp = domain
values(temp)[old_idx] = 2
par(mar = c(1,1,1,1))
plot(temp, axes = F, legend = F, col = c('white', 'green'))
plot(raster::rasterToPolygons(domain), 
     axes = F,
     border = 'grey',
     add = T)
```

### Example Pareto Frontier

As an example, we can trace a Pareto Frontier under two objectives, conservation value (maximize) and opportunity cost (minimize) under fixed total area (A = 0.3) and compactness constraints. This is done by solving the ILP model under varying conservation constraints. The solutions are plotted below, color-coded by the compactness constraint.

```{r, echo = F}
res_df = data.frame()
eps_spp = 0.01
compactness_scen = c(0.01,0.75,1)
A = 0.3
for(b in 1:length(compactness_scen)){
  temp_spp = 0.01
  temp_res = do_glpkAPI(area_constraint = A, 
                        boundary_constraint = A * compactness_scen[b],
                        species_constraint = temp_spp,
                        gap = 0.01, 
                        time_limit = Inf, bound = NA)
  status = temp_res$output_code
  
  while(!(status %in% c("[1] 10", "[1] 9") )){
    temp_chosen_pus = which(temp_res$x[1:n_pus] == 1)
    temp_spp = sum(df$spp[temp_chosen_pus]) / sum(df$spp)
    temp_objval = temp_res$objval
    temp_compact = sum(temp_res$x[-1*(1:n_pus)]) / M
    temp_t = temp_res$time
    
    res_df = rbind(res_df, 
                   data.frame('B' = compactness_scen[b],
                              'compact' = temp_compact,
                              'spp' = temp_spp, 
                              'cost' = temp_objval,
                              'time' = temp_t))
    
    temp_spp = temp_spp + eps_spp
    
    temp_res = do_glpkAPI(area_constraint = A, 
                          boundary_constraint = A * compactness_scen[b],
                          species_constraint = temp_spp,
                          gap = 0.01, 
                          time_limit = 10, 
                          bound = NA)
    status = temp_res$output_code
  }
}
```

```{r, echo = F}
plot(cost ~ spp, data = res_df, subset = (B == 0.01),
     pch = 16, las = 1, type = 'b',
     xlab = 'Conservation Value', ylab = 'Opportunity Cost',
     xlim = c(0,0.7), ylim = c(0,600))
lines(cost ~ spp, data = res_df, subset = (B == 0.75), col = 'red')
points(cost ~ spp, data = res_df, subset = (B == 0.75), col = 'red', pch = 16)
lines(cost ~ spp, data = res_df, subset = (B == 1), col = 'blue')
points(cost ~ spp, data = res_df, subset = (B == 1), col = 'blue', pch = 16)
legend('topleft', legend = paste(c('Low', 'Mid', 'High'), 'Compactness'),
       pch = 16, col = c('black', 'red', 'blue'))
```
