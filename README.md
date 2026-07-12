# scRNA-seq clustering and feature selection for cell type differentiation

Note: This is part of the [MITx 6.419x course](https://www.edx.org/learn/data-analysis/massachusetts-institute-of-technology-data-analysis-statistical-modeling-and-computation-in-applications). The datasets are not included in this repository because of the size restriction. Refer to `problem 2.ipynb` for the main analysis of this project.

dataset organization:
- gene_names
- p1
    - X.npy
    - y.npy
- p2_evaluation
    - X_test.npy
    - X_train.npy
    - y_test.npy
    - y_train.npy
- p2_evaluation_reduced
    - X_test.npy
    - X_train.npy
    - y_test.npy
    - y_train.npy
- p2_unsupervised
    - X.npy
- p2_unsupervised_reduced
    - X.npy

This project analyzed single-cell RNA sequencing data to identify distinct cell populations and informative genes. I first used dimensional reduction and hierarchical clustering to divide cells into major classes and their subtypes. Then, I treated the cluster assignments as labels for regularized multiclass logistic regression, using the regression coefficients to select the 100 most informative genes. Finally, I evaluated these genes on an independent dataset and compare their classification performance with random and highest-variance gene selections.

## Dimensional reduction
- The original scRNA-seq matrix contained 2169 cells x 45768 genes. I first log-transformed the data using $x' = \log_2(x+1)$ to reduce the influence of highly abundant genes.

- Applied **Principal Component Analysis (PCA)** and retained the first 50 principal components. I also used **multidimensional scaling (MDS)** and **t-distributed Stochastic Neighbor Embedding (t-SNE)** to visualize the cells in 2D. Both PCA and MDS produced 3 major classes of cells (see Figure 1 & 2). 

- In particular, t-SNE was applied to the 50-dimensional PCA representation to produce a 2D visualization. The clustering was performed using the 50-dimensional PCA representation, but I presented the clusters/labels on top of the t-SNE plot. 

## Hierarchical clustering
- Performed **agglomerative hierarchical clustering** using the 50-dimensional PCA representation. Agglomerative clustering treats every cell as its own cluster and then repeatedly merging the most similar clusters until the desired number of clusters remains.
    - I used Ward linkage, which merged clusters in a way that minimizes the within-cluster variance.

- To determine the number of clusters and subtypes, I evaluated candidate k values (represented the number of clusters) using silhouette scores: larger values indicated that cells were generally closer to their own cluster than to neighboring clusters.
    - The silhouette analysis favored 2 major clusters numerically (see Figure 3). However, I still used 3 major clusters as described in the guildlines (excitatory neurons, inhibitory neurons, and non-neurons). They are labeled as `M1`(major cluster 1), `M2`(major cluster 2), and `M3`(major cluster 3).
    - The number of subtypes was selected separately for each major cluster: 11 subtypes in M1, 4 subtypes in M2, 6 subtypes in M3.
        - Note that in Figure 4, the optimal k value for major cluster 1 (M1) was 18. However, the plot showed tiny improvement of silhouette score over k = 11, and to avoid over-fragmented subtypes, I chose k = 11 instead of 18.
- The final hierarchical clustering result was present in Figure 5.

## Multiclass logistic regression
- I treated the subtype assignments from hierarchical clustering as target labels for supervised learning. Because the target contains multiple subtypes, I used a one-versus-rest strategy. For each subtype, a binary logistic regression classifier distinguished that subtype from all other subtypes. The predicted class for a cell is the subtype whose classifier produced the strongest prediction (highest probability).
    - As for regularization, I chose L1 because it encourages many gene coefficients to become 0, because the goal was to select a small number (100) of informative genes from the original 45768 features.

- I divided `p2_unsupervised` data into training and validation sets. Within the training set, I used five-fold stratified cross-validation to select the regularization parameter $C$. I tested $C \in \{10^{-3}, 10^{-2}, 10^{-1}, 1, 10, 100\}$.

- The selected value was $C$ = 0.1 based on the Figure 6, with a mean cross-validation accuracy of approximately 0.9452. 

- The final model was refitted using the full training set with the selected value of $C$, and then evaluated on the held-out validation set. The validation accuracy was approximately 0.9447.
- After fitting the model, I then examined the coefficient matrix. For each gene, I calculated its maximum absolute coefficient across all subtype classifiers: $I_j = \max_k \left|\beta_{kj}\right|$, where $I_j$ is the importance of gene $j$, and $\beta_{kj}$ is the coefficient for gene $j$ in the classifier for subtype $k$. The 100 genes with the largest importance values were selected. They represented features that distinguish all subtypes. 

- To evaluate whether the logistic-selected genes generalized to another dataset, I used the `p2_evaluation` training and test sets. A logistic regression classifier was trained by cross-validation to classify the cell labels with logistic-selected genes as predictors, on the evaluation training data and evaluated once on the evaluation test data. 
    - After the cross-validation, I selected optimal $C$ = 1.0.
    - The logistic-selected genes achieved approximately 85.38% test accuracy.
    - For comparison, I also evaluated two baselines: 
        - Randomly selected genes: 41.88%
        - Highest-variance genes: 92.60%
        - All three methods use exactly 100 genes, and $C$ = 1.0.

- The random feature baseline performed substantially worse, indicating it didn't capture meaningful gene-expression signal. The logistic regression selected genes also performed much better than random genes, however, the highest-variance genes achieved the best performance on the evaluation dataset. 
- The variance histogram in Figure 7 showed that the logistic regression selected genes generally had lower variance than the highest-variance genes. This indicates that coefficient based feature selection identified genes were not simply the most variable genes. Nevertheless, for this evaluation dataset, the highest-variance genes transferred better to the classification task than the genes selected from the unsupervised clustering model.
