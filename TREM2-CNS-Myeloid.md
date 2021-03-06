```R
library(Seurat)
sce.merge <- CreateSeuratObject(counts = count,
                                min.cells = 0,
                                min.features = 0,
                                project = "sce.merge")

## Organize and scale data for PCA analysis
sce.merge <- NormalizeData(sce.merge, normalization.method = "LogNormalize", scale.factor = 10000)
sce.merge <- FindVariableFeatures(sce.merge,selection.method = "vst", nfeatures = 5000)
sce.merge <- ScaleData(object = sce.merge,
                       vars.to.regress = c('nCount_RNA'),
                       model.use = 'linear',
                       use.umi = FALSE)
sce.merge <- RunPCA(object = sce.merge, verbose = FALSE)
ElbowPlot(sce.merge)

## Umap
set.seed(666) 
sce.merge <- RunUMAP(object = sce.merge, dims = 1:19, do.fast = TRUE)
sce.merge <- FindNeighbors(sce.merge, dims = 1:19)
sce.merge <- FindClusters(sce.merge, resolution = 0.5)
p1 <- DimPlot(sce.merge,reduction = "umap",label=T)

## Find markers for each cluster
sce.markers <- FindAllMarkers(object = sce.merge, only.pos = TRUE, min.pct = 0.25, thresh.use = 0.25)
write.csv(sce.markers, file = "sce.markers.csv")

## SingleR
library(SingleR)
sce_for_SingleR <- GetAssayData(sce.merge, slot="data")
clusters=sce.merge@meta.data$seurat_clusters
ref <- MouseRNAseqData()
df <- SingleR(test = sce_for_SingleR, ref = ref, labels = ref$label.fine,
              method = "cluster", clusters = clusters, 
              assay.type.test = "logcounts", assay.type.ref = "logcounts")
cellType=data.frame(ClusterID = levels(factor(sot@meta.data$seurat_clusters)),
                    HumanRNA=df$labels)
refmm <- ImmGenData()
dfmm <- SingleR(test = sce_for_SingleR, ref = refmm, labels = refmm$label.main,
                method = "cluster", clusters = clusters, 
                assay.type.test = "logcounts", assay.type.ref = "logcounts")
cellType=data.frame(ClusterID = levels(factor(sce.merge@meta.data$seurat_clusters)),
                    mouseRNA=df$labels,
                    mouseImmu=dfmm$labels )
sce.merge@meta.data$singleR=cellType[match(clusters,cellType$ClusterID),'mouseRNA']
p2 <- DimPlot(sce.merge,reduction = "umap",label=T,group.by  ="singleR",split.by ='SampleName')
p3 <- DimPlot(sce.merge,reduction = "umap",label=T,group.by = 'singleR')

## Add corresponding disease type info to samples
write.table(sce.merge@meta.data, "sce.mergemeta.txt", quote = FALSE, sep = "\t")
metamerge <- read.table("sce.mergemeta.dis.txt",header = T,row.names = 1)
sce.merge@meta.data <- metamerge
p4 <- DimPlot(sce.merge,reduction ="umap",group.by = "Disease",label=F)

## Show FeaturePlot and ViolinPlot
library(tidyverse)
library(dplyr)
library(patchwork)
load(file = "sce.dis.clean.RData")
p5 <- FeaturePlot(sce.dis.clean, features = "Trem2", split.by = "Genotype",pt.size = 0.5)
p6 <- FeaturePlot(sce.dis.clean, features = "Atp6v0a2", split.by = "Cell_label", pt.size = 0.5)
p7 <- VlnPlot(sce.dis.clean, features = c("Trem2"),group.by = "Cell_label",pt.size = 0.2,cols = my_cols)

## Show Wt data
metamerge <- read.table("sce.mergemeta.dis.wt.clean.txt",header = T,row.names = 1)
metamerge$cellname <- rownames(metamerge)
sce.wt.clean.new <-sce.merge[, as.character(metamerge$cellname)]
sce.wt.clean.new@meta.data <- metamerge
my_cols <- c('1'='#2FF18B','16'='#31C53F','4'='#aeadb3','6'='#1FA195','2'='#28CECA',
             '9'='#CCB1F1','3'='#ff9a36','11'='#25aff5','5'='#D4D915','7'='#F68282',
             '0'='#B95FBB')
save(sce.wt.clean.new,file = "sce.wt.clean.new.RData")

## Compare microglial clusters
load(file = "sce.wt.clean.new.RData")
Mi_4.sig <- FindMarkers(sce.wt.clean.new,
                        ident.1 = 4,
                        ident.2 = c(4,6))
write.csv(Mi_4.sig, file = "Mi_4.sig.csv")

## Pseudotime analysis
library(monocle)
library(tidyverse)
load(file = "sce.wt.clean.new.RData")
data <- as(as.matrix(sce.wt.clean.new@assays$RNA@counts), 'sparseMatrix')
pData <- data.frame(sce.wt.clean.new@meta.data)
pData$Clusters <- str_c("Cluster",pData$seurat_clusters)
pd <- new('AnnotatedDataFrame', data = pData)
fData <- data.frame(gene_short_name = row.names(data), row.names = row.names(data))
fd <- new('AnnotatedDataFrame', data = fData)
mycds <- newCellDataSet(data,
                        phenoData = pd,
                        featureData = fd,
                        expressionFamily = negbinomial.size())
mycds <- estimateSizeFactors(mycds)
save(mycds,file = "sce.mergemeta.dis.clean.sci.mic46.monocle1.RData")
mycds <- estimateDispersions(mycds, cores=6, relative_expr = TRUE)
disp_table <- dispersionTable(mycds)
disp.genes <- subset(disp_table, 
                     mean_expression >= 0.5 & dispersion_empirical >= 0.1 * dispersion_fit)$gene_id
mycds <- setOrderingFilter(mycds, disp.genes)
plot_ordering_genes(mycds)
mycds <- reduceDimension(mycds, max_components = 2)
save(mycds,file = "sce.mergemeta.dis.clean.sci.mic46.monocle2.RData")
load(file = "sce.mergemeta.dis.clean.sci.mic46.monocle2.RData")
mycds <- orderCells(mycds,reverse = FALSE)
my_cols <- c('Wt_1'='#2FF18B','Wt_16'='#31C53F','Wt_4'='#aeadb3','Wt_6'='#1FA195','Wt_2'='#28CECA',
             'Wt_9'='#CCB1F1','Wt_3'='#ff9a36','Wt_11'='#25aff5','Wt_5'='#D4D915','Wt_7'='#F68282',
             'Wt_0'='#B95FBB','Tm2_1'='#3d7356','Tm2_16'='#31C55F','Tm2_4'='#aeadb3','Tm2_6'='#1FA195','Tm2_2'='#28CECB',
             'Tm2_9'='#CCB1F3','Tm2_3'='#b58250','Tm2_11'='#467f9c','Tm2_5'='#949647','Tm2_7'='#F68292',
             'Tm2_0'='#B95FBC')
plot_cell_trajectory(mycds, cell_size = 2, color_by = "Wt_tm2") + facet_grid(. ~Cell_label) + scale_color_manual(breaks = c("X", "Y", "Z"),  values=my_cols)
plot_cell_trajectory(mycds, cell_size = 1, color_by = "Wt_tm2") + scale_color_manual(breaks = c("X", "Y", "Z"),  values=my_cols)
plot_cell_trajectory(mycds, color_by = "Disease")
plot_cell_trajectory(mycds, color_by = "Pseudotime",cell_size = 2)
```

