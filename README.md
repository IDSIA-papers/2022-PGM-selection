# Bounding Counterfactuals under Selection Bias

This bundle contains the manuscript submited to the PGM2022 and entitled  "Bounding Counterfactuals under Selection Bias".
The organisation  is the following:

- _example_: code for replicating the examples in the paper and an additional toy example.
- _experiments_: Java files for replicating the experiments.
- _lib_: packages needed for running the code.
- _models_: set of structural causal models in UAI format considered in the experimentation.
- _pom.xml_: maven configuration file required for building a jar file with the sources.



## Setup

The code has been tested with **Java openjdk 12.0.1**. For checking The Java version running in your system use the
following command:

```bash
$ java --version
```

```
openjdk 12.0.1 2019-04-16
OpenJDK Runtime Environment (build 12.0.1+12)
OpenJDK 64-Bit Server VM (build 12.0.1+12, mixed mode, sharing)
```


## Running EMCC under selection bias

Here we present a minimal example in java for applying the method. This requires either the packages in `lib` folder, or importing
CREDICI 0.1.5 (through maven). The full code is available at `examples/ToyExample.java`

First, we define two DAGs for characterizing a SCM. First the DAG only with the endogenous variables $X,Y$ and then the one
also including the exogenous variables $U,V$.

```java
// Variables IDs
int X = 0, Y = 1;
int U = 2, V = 3;

// Define endogenous DAG and complete DAG of the SCM
SparseDirectedAcyclicGraph endoDAG = DAGUtil.build("(0,1)");
SparseDirectedAcyclicGraph causalDAG = DAGUtil.build("(0,1),(2,0),(3,1)");
```

Now we can create an object representing the SCM with a conservative specification and random PMF for the exogenous variables.

```java
// Create the SCM with random probabilities for the exogenous variables
StructuralCausalModel model = CausalBuilder.of(endoDAG, 2).setCausalDAG(causalDAG).build();
model.fillExogenousWithRandomFactors(5);

logger.info("Model structure: "+model.getNetwork());
logger.info("Model parameters: "+model);
```

Now, the dataset (before selection) is sampled.

```java
// Sample a complete dataset
TIntIntMap[] data = model.samples(1000, model.getEndogenousVars());
// Empirical endogenous distribution from the data
HashMap empiricalDist = DataUtil.getEmpiricalMap(model, data);
empiricalDist = FactorUtil.fixEmpiricalMap(empiricalDist,6);

logger.info("Sampled complete data with size: "+data.length);
```

For the running example, let's consider that the joint states $(X=0,Y=0)$ and $(X=1,Y=1)$ are not available. So these are defined
in the array `hidden_conf`. The parents of the selector variable are `X` and `Y`. Now we can create the extended model for bias
selection and the selected data as follows.

```java    
// Non available configurations for X,Y
int[][] hidden_conf = new int[][]{{0,0},{1,1}};
int[] Sparents = new int[]{X,Y};

// Model with Selection Bias structure
StructuralCausalModel modelBiased = SelectionBias.addSelector(model, Sparents, hidden_conf);
int selectVar = ArraysUtil.difference(modelBiased.getEndogenousVars(), model.getEndogenousVars())[0];

// Biased data
TIntIntMap[] dataBiased = SelectionBias.applySelector(data, modelBiased, selectVar);
```

Finally we can run EMCC and obtain a set of precise SCMs from which we can bound any query.

```java
logger.info("Running EMCC with selected data.");

// Learn the model
List endingPoints = SelectionBias.runEM(modelBiased, selectVar, dataBiased, maxIter, executions);

// Run inference
CausalMultiVE multiInf = new CausalMultiVE(endingPoints);
VertexFactor p = (VertexFactor) multiInf.probNecessityAndSufficiency(X, Y);
logger.info("PNS: ["+p.getData()[0][0][0]+", "+p.getData()[0][1][0]+"]");
```
