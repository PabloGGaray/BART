# PyMC-BART Code Profiling

Welcome to the PyMC-BART line profiler.

## Getting started

Miniconda is used to setup the environment.

Once miniconda is installed, there is an `environment.yml` file to create an environment with all the dependencies needed to run the line profiling tests.

In order to provide line-by-line profiling, specific functions and methods have been decorated (see the results section) in the PyMC-BART codebase have been annotated with the `@profile` decorator. These annotations have been done on the branch `git+https://github.com/GStechschulte/pymc-bart.git@line-profiler`. Thus, the `environment.yml` installs PyMC-BART from this branch.

```bash
conda env create -f environment.yml
conda activate bart-line-profiler
```

To run the benchmarks

```bash
bash benchmark.sh
```

This will run line profiling benchmarks on the models in the `case_studies` directory with a variety of different hyperparameter values and save the results in the `results` directory. The only common hyperparameter between all the models is the number of iterations, which is set to 500.

## Results

To view the results of any one result file, run

```bash
python -m line_profiler results/<filename>.lprof
```

There are several results within one `.prof` file. This is because several methods and functions have been decorated with the `@profile` decorator. In our case, we are interested in profiling the `.astep` method (and the subsequent call stack) of the `PGBART` sampler as this method represents the main entry point of the particle sampler.

Overall, eight functions and or methods are profiled.

```bash
└── astep
    ├── sample_tree
    │   └── grow_tree
    │       └── draw_leaf_value
    ├── update_weight
    │   └── logp
    └── resample
        └── systematic
```

#### Biking

As there is no existing tool to aggregate the results of **multiple** `.lprof` files, only the results of the biking model with 200 trees and 60 particles is shown. The results of the remaining case studies were analyzed individually and in a similar manner. The line profiling results of the remaining case studies showed similar `Hits`, `Time`, `Per Hit`, `% Time` patterns as the results shown below.

The listing below can be reproduced by executing `python -m line_profiler results/biking_200_60.lprof`

```bash
Total time: 354.818 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: astep at line 226

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   226                                               @profile
   227                                               def astep(self, _):
   228       500       1622.0      3.2      0.0          variable_inclusion = np.zeros(self.num_variates, dtype="int")
   229                                           
   230       500       6000.0     12.0      0.0          upper = min(self.lower + self.batch[~self.tune], self.m)
   231       500        198.0      0.4      0.0          tree_ids = range(self.lower, upper)
   232       500        179.0      0.4      0.0          self.lower = upper if upper < self.m else 0
   233                                           
   234      1000        373.0      0.4      0.0          for odim in range(self.trees_shape):
   235     10500       2793.0      0.3      0.0              for tree_id in tree_ids:
   236     10000       2655.0      0.3      0.0                  self.iter += 1
   237                                                           # Compute the sum of trees without the old tree that we are attempting to replace
   238     10000       7713.0      0.8      0.0                  self.sum_trees_noi[odim] = (
   239     10000      75551.0      7.6      0.0                      self.sum_trees[odim] - self.all_particles[odim][tree_id].tree._predict()
   240                                                           )
   241                                                           # Generate an initial set of particles
   242                                                           # at the end we return one of these particles as the new tree
   243     10000    2895929.0    289.6      0.8                  particles = self.init_particles(tree_id, odim)
   244                                           
   245     59197      10492.0      0.2      0.0                  while True:
   246                                                               # Sample each particle (try to grow each tree), except for the first one
   247     59197       7384.0      0.1      0.0                      stop_growing = True
   248   3551820     816730.0      0.2      0.2                      for p in particles[1:]:
   249   6985246  119835232.0     17.2     33.8                          if p.sample_tree(
   250   3492623     378732.0      0.1      0.1                              self.ssv,
   251   3492623     367657.0      0.1      0.1                              self.available_predictors,
   252   3492623     343098.0      0.1      0.1                              self.prior_prob_leaf_node,
   253   3492623     382974.0      0.1      0.1                              self.X,
   254   3492623     324124.0      0.1      0.1                              self.missing_data,
   255   3492623     737776.0      0.2      0.2                              self.sum_trees[odim],
   256   3492623     642514.0      0.2      0.2                              self.leaf_sd[odim],
   257   3492623     387686.0      0.1      0.1                              self.m,
   258   3492623     370114.0      0.1      0.1                              self.response,
   259   3492623     423368.0      0.1      0.1                              self.normal,
   260   3492623     373876.0      0.1      0.1                              self.leaves_shape,
   261                                                                   ):
   262   1958508  209543773.0    107.0     59.1                              self.update_weight(p, odim)
   263   3492623     642996.0      0.2      0.2                          if p.expansion_nodes:
   264   2580280     318601.0      0.1      0.1                              stop_growing = False
   265     59197      10681.0      0.2      0.0                      if stop_growing:
   266     10000       1784.0      0.2      0.0                          break
   267                                           
   268                                                               # Normalize weights
   269     49197     795030.0     16.2      0.2                      normalized_weights = self.normalize(particles[1:])
   270                                           
   271                                                               # Resample
   272     49197   14135883.0    287.3      4.0                      particles = self.resample(particles, normalized_weights)
   273                                           
   274     10000     142056.0     14.2      0.0                  normalized_weights = self.normalize(particles)
   275                                                           # Get the new particle and associated tree
   276     20000     114749.0      5.7      0.0                  self.all_particles[odim][tree_id], new_tree = self.get_particle_tree(
   277     10000       1187.0      0.1      0.0                      particles, normalized_weights
   278                                                           )
   279                                                           # Update the sum of trees
   280     10000      60039.0      6.0      0.0                  new = new_tree._predict()
   281     10000      19010.0      1.9      0.0                  self.sum_trees[odim] = self.sum_trees_noi[odim] + new
   282                                                           # To reduce memory usage, we trim the tree
   283     10000      93182.0      9.3      0.0                  self.all_trees[odim][tree_id] = new_tree.trim()
   284                                           
   285     10000       1889.0      0.2      0.0                  if self.tune:
   286                                                               # Update the splitting variable and the splitting variable sampler
   287     10000       2080.0      0.2      0.0                      if self.iter > self.m:
   288      9800     101023.0     10.3      0.0                          self.ssv = SampleSplittingVariable(self.alpha_vec)
   289                                           
   290     29190      39311.0      1.3      0.0                      for index in new_tree.get_split_variables():
   291     19190       8570.0      0.4      0.0                          self.alpha_vec[index] += 1
   292                                           
   293                                                               # update standard deviation at leaf nodes
   294     10000       2163.0      0.2      0.0                      if self.iter > 2:
   295      9998      81526.0      8.2      0.0                          self.leaf_sd[odim] = self.running_sd[odim].update(new)
   296                                                               else:
   297         2     305928.0 152964.0      0.1                          self.running_sd[odim].update(new)
   298                                           
   299                                                           else:
   300                                                               # update the variable inclusion
   301                                                               for index in new_tree.get_split_variables():
   302                                                                   variable_inclusion[index] += 1
   303                                           
   304       500        125.0      0.2      0.0          if not self.tune:
   305                                                       self.bart.all_trees.append(self.all_trees)
   306                                           
   307       500        282.0      0.6      0.0          stats = {"variable_inclusion": variable_inclusion, "tune": self.tune}
   308       500       1161.0      2.3      0.0          return self.sum_trees, [stats]
```

Where the majority of the time is spent in `p.sample_tree`, `self.update_weight`, and `self.resample`. Within `p.sample_tree`

```bash
Total time: 106.285 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: sample_tree at line 56

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    56                                               @profile
    57                                               def sample_tree(
    58                                                   self,
    59                                                   ssv,
    60                                                   available_predictors,
    61                                                   prior_prob_leaf_node,
    62                                                   X,
    63                                                   missing_data,
    64                                                   sum_trees,
    65                                                   leaf_sd,
    66                                                   m,
    67                                                   response,
    68                                                   normal,
    69                                                   shape,
    70                                               ) -> bool:
    71   3492623     340268.0      0.1      0.3          tree_grew = False
    72   3492623     508034.0      0.1      0.5          if self.expansion_nodes:
    73   2866243     475991.0      0.2      0.4              index_leaf_node = self.expansion_nodes.pop(0)
    74                                                       # Probability that this node will remain a leaf node
    75   2866243     565298.0      0.2      0.5              prob_leaf = prior_prob_leaf_node[get_depth(index_leaf_node)]
    76                                           
    77   2866243    1639673.0      0.6      1.5              if prob_leaf < np.random.random():
    78   3942482   98153863.0     24.9     92.3                  idx_new_nodes = grow_tree(
    79   1971241     230667.0      0.1      0.2                      self.tree,
    80   1971241     231216.0      0.1      0.2                      index_leaf_node,
    81   1971241     199968.0      0.1      0.2                      ssv,
    82   1971241     178471.0      0.1      0.2                      available_predictors,
    83   1971241     182496.0      0.1      0.2                      X,
    84   1971241     176217.0      0.1      0.2                      missing_data,
    85   1971241     181339.0      0.1      0.2                      sum_trees,
    86   1971241     186067.0      0.1      0.2                      leaf_sd,
    87   1971241     185867.0      0.1      0.2                      m,
    88   1971241     204993.0      0.1      0.2                      response,
    89   1971241     173099.0      0.1      0.2                      normal,
    90   1971241     178087.0      0.1      0.2                      shape,
    91                                                           )
    92   1971241     264774.0      0.1      0.2                  if idx_new_nodes is not None:
    93   1958508     369831.0      0.2      0.3                      self.expansion_nodes.extend(idx_new_nodes)
    94   1958508     216406.0      0.1      0.2                      tree_grew = True
    95                                           
    96   3492623    1442335.0      0.4      1.4          return tree_grew
```

As expected, when sampling a tree, the majority of the time is then spent calling `grow_tree`.

```bash
Total time: 80.0609 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: grow_tree at line 473

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   473                                           @profile
   474                                           def grow_tree(
   475                                               tree,
   476                                               index_leaf_node,
   477                                               ssv,
   478                                               available_predictors,
   479                                               X,
   480                                               missing_data,
   481                                               sum_trees,
   482                                               leaf_sd,
   483                                               m,
   484                                               response,
   485                                               normal,
   486                                               shape,
   487                                           ):
   488   1971241     710679.0      0.4      0.9      current_node = tree.get_node(index_leaf_node)
   489   1971241     253434.0      0.1      0.3      idx_data_points = current_node.idx_data_points
   490                                           
   491   1971241    2195372.0      1.1      2.7      index_selected_predictor = ssv.rvs()
   492   1971241     271732.0      0.1      0.3      selected_predictor = available_predictors[index_selected_predictor]
   493   3942482    1414023.0      0.4      1.8      idx_data_points, available_splitting_values = filter_missing_values(
   494   1971241    3167712.0      1.6      4.0          X[idx_data_points, selected_predictor], idx_data_points, missing_data
   495                                               )
   496                                           
   497   1971241     268756.0      0.1      0.3      split_rule = tree.split_rules[selected_predictor]
   498                                           
   499   1971241    2513708.0      1.3      3.1      split_value = split_rule.get_split_value(available_splitting_values)
   500                                           
   501   1971241     288121.0      0.1      0.4      if split_value is None:
   502     12733       3889.0      0.3      0.0          return None
   503                                           
   504   1958508    2628603.0      1.3      3.3      to_left = split_rule.divide(available_splitting_values, split_value)
   505   1958508    4765818.0      2.4      6.0      new_idx_data_points = idx_data_points[to_left], idx_data_points[~to_left]
   506                                           
   507   1958508     241881.0      0.1      0.3      current_node_children = (
   508   1958508     743359.0      0.4      0.9          get_idx_left_child(index_leaf_node),
   509   1958508     587084.0      0.3      0.7          get_idx_right_child(index_leaf_node),
   510                                               )
   511                                           
   512   1958508     366340.0      0.2      0.5      if response == "mix":
   513                                                   response = "linear" if np.random.random() >= 0.5 else "constant"
   514                                           
   515   5875524    1295806.0      0.2      1.6      for idx in range(2):
   516   3917016     492628.0      0.1      0.6          idx_data_point = new_idx_data_points[idx]
   517   7834032   28268780.0      3.6     35.3          node_value, linear_params = draw_leaf_value(
   518   3917016    5501689.0      1.4      6.9              y_mu_pred=sum_trees[:, idx_data_point],
   519   3917016    4444059.0      1.1      5.6              x_mu=X[idx_data_point, selected_predictor],
   520   3917016     515006.0      0.1      0.6              m=m,
   521   3917016    4413109.0      1.1      5.5              norm=normal.rvs() * leaf_sd,
   522   3917016     456504.0      0.1      0.6              shape=shape,
   523   3917016     439600.0      0.1      0.5              response=response,
   524                                                   )
   525                                           
   526   7834032    5914257.0      0.8      7.4          new_node = Node.new_leaf_node(
   527   3917016     431503.0      0.1      0.5              value=node_value,
   528   3917016     599473.0      0.2      0.7              nvalue=len(idx_data_point),
   529   3917016     432604.0      0.1      0.5              idx_data_points=idx_data_point,
   530   3917016     412419.0      0.1      0.5              linear_params=linear_params,
   531                                                   )
   532   3917016    3368826.0      0.9      4.2          tree.set_node(current_node_children[idx], new_node)
   533                                           
   534   1958508    1384203.0      0.7      1.7      tree.grow_leaf_node(current_node, selected_predictor, split_value, index_leaf_node)
   535   1958508    1269953.0      0.6      1.6      return current_node_children
```

Within `grow_tree` the timing is more spread out. Still, the majority is spent calling `draw_leaf_value`.

```bash
Total time: 15.4686 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: draw_leaf_value at line 545

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   545                                           @profile
   546                                           def draw_leaf_value(
   547                                               y_mu_pred: npt.NDArray[np.float_],
   548                                               x_mu: npt.NDArray[np.float_],
   549                                               m: int,
   550                                               norm: npt.NDArray[np.float_],
   551                                               shape: int,
   552                                               response: str,
   553                                           ) -> Tuple[npt.NDArray[np.float_], Optional[npt.NDArray[np.float_]]]:
   554                                               """Draw Gaussian distributed leaf values."""
   555   3917016     428582.0      0.1      2.8      linear_params = None
   556   3917016    1168043.0      0.3      7.6      mu_mean = np.empty(shape)
   557   3917016     620698.0      0.2      4.0      if y_mu_pred.size == 0:
   558     50545      23182.0      0.5      0.1          return np.zeros(shape), linear_params
   559                                           
   560   3866471     517915.0      0.1      3.3      if y_mu_pred.size == 1:
   561     48983     181726.0      3.7      1.2          mu_mean = np.full(shape, y_mu_pred.item() / m) + norm
   562   3817488     734071.0      0.2      4.7      elif y_mu_pred.size < 3 or response == "constant":
   563   3817488   10285725.0      2.7     66.5          mu_mean = fast_mean(y_mu_pred) / m + norm
   564                                               else:
   565                                                   mu_mean, linear_params = fast_linear_fit(x=x_mu, y=y_mu_pred, m=m, norm=norm)
   566                                           
   567   3866471    1508660.0      0.4      9.8      return mu_mean, linear_params
```

Inside of `draw_leaf_value` the majority of the time is spent computing the mean of the leaf node by calling `fast_mean`. Interestingly, `fast_mean` is JIT compiled using `numba`. Going back up the call stack to `self.update_weight` 

```bash
Total time: 205.514 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: update_weight at line 379

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   379                                               @profile
   380                                               def update_weight(self, particle: ParticleTree, odim: int) -> None:
   381                                                   """
   382                                                   Update the weight of a particle.
   383                                                   """
   384                                           
   385   1968508     250828.0      0.1      0.1          delta = (
   386   3937016    9704852.0      2.5      4.7              np.identity(self.trees_shape)[odim][:, None, None]
   387   1968508   11168149.0      5.7      5.4              * particle.tree._predict()[None, :, :]
   388                                                   )
   389                                           
   390   1968508  183663423.0     93.3     89.4          new_likelihood = self.likelihood_logp((self.sum_trees_noi + delta).flatten())
   391   1968508     726430.0      0.4      0.4          particle.log_weight = new_likelihood
```

The majority of the time is spent calculating the log likelihood of the new particle tree. The requires calling `logp`

```bash
Total time: 0.305182 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: logp at line 731

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   731                                           @profile
   732                                           def logp(point, out_vars, vars, shared):  # pylint: disable=redefined-builtin
   733                                               """Compile PyTensor function of the model and the input and output variables.
   734                                           
   735                                               Parameters
   736                                               ----------
   737                                               out_vars: List
   738                                                   containing :class:`pymc.Distribution` for the output variables
   739                                               vars: List
   740                                                   containing :class:`pymc.Distribution` for the input variables
   741                                               shared: List
   742                                                   containing :class:`pytensor.tensor.Tensor` for depended shared data
   743                                               """
   744         1       4205.0   4205.0      1.4      out_list, inarray0 = join_nonshared_inputs(point, out_vars, vars, shared)
   745         1     300976.0 300976.0     98.6      function = pytensor_function([inarray0], out_list[0])
   746         1          1.0      1.0      0.0      function.trust_input = True
   747         1          0.0      0.0      0.0      return function
```

Which in turn relies on calling `pytensor_function`. Again, going back up the call stack to `self.resample`

```bash
Total time: 12.2683 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: resample at line 320

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   320                                               @profile
   321                                               def resample(
   322                                                   self, particles: List[ParticleTree], normalized_weights: npt.NDArray[np.float_]
   323                                               ) -> List[ParticleTree]:
   324                                                   """
   325                                                   Use systematic resample for all but the first particle
   326                                           
   327                                                   Ensure particles are copied only if needed.
   328                                                   """
   329     49197     639961.0     13.0      5.2          new_indices = self.systematic(normalized_weights) + 1
   330     49197       6527.0      0.1      0.1          seen: List[int] = []
   331     49197       5902.0      0.1      0.0          new_particles: List[ParticleTree] = []
   332   2951820     598776.0      0.2      4.9          for idx in new_indices:
   333   2902623     953584.0      0.3      7.8              if idx in seen:
   334   1172744    8614819.0      7.3     70.2                  new_particles.append(particles[idx].copy())
   335                                                       else:
   336   1729879     217679.0      0.1      1.8                  new_particles.append(particles[idx])
   337   1729879     202746.0      0.1      1.7                  seen.append(idx)
   338                                           
   339     49197    1004893.0     20.4      8.2          particles[1:] = new_particles
   340                                           
   341     49197      23392.0      0.5      0.2          return particles
```

most of the time is spent inside the `for` loop appending to `new_particles` and evaluating control flow statements. Additionally, `self.systematic` takes the fourth most time in the `resample` method. It can be seen below that the majority of the time inside of `self.systematic` is spent computing the `inverse_cdf`.

```bash
Total time: 0.533195 s
File: /Users/gabestechschulte/miniforge3/envs/bart-line-profiler/lib/python3.12/site-packages/pymc_bart/pgbart.py
Function: systematic at line 356

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   356                                               @profile
   357                                               def systematic(self, normalized_weights: npt.NDArray[np.float_]) -> npt.NDArray[np.int_]:
   358                                                   """
   359                                                   Systematic resampling.
   360                                           
   361                                                   Return indices in the range 0, ..., len(normalized_weights)
   362                                           
   363                                                   Note: adapted from https://github.com/nchopin/particles
   364                                                   """
   365     59197      10446.0      0.2      2.0          lnw = len(normalized_weights)
   366     59197     255931.0      4.3     48.0          single_uniform = (self.uniform.rvs() + np.arange(lnw)) / lnw
   367     59197     266818.0      4.5     50.0          return inverse_cdf(single_uniform, normalized_weights)
```