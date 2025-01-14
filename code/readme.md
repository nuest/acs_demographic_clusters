# Analysis code for "Studying Neighborhoods Using Uncertain Data from the American Community Survey"

This document outlines the procedeures necessary to reproduce the analysis in 
[Studying Neighborhoods Using Uncertain Data from the American Community Survey: A Contextual Approach](https://doi.org/10.1080/00045608.2015.1052335) by Seth Spielman and Alex Singleton.  All of the data used below
[downloadable on openICPSR](http://doi.org/10.3886/E41329V2).  We refer readers 
to the full paper for the motivations behind out various methodological choices.  

```{r}
#Load libraries
library(cluster)
library(ggplot2)
library(maptools)
library(rgdal)
library(reshape2)
library(plyr)
library(xtable)

set.seed(7777) #reproducibality
```

## Loading and Preparing Census Data

The data used in this analysis was downloaded from [Social Explorer](http://www.socialexplorer.com).  Social Explorer provides raw data from the ACS and data that they have processed and reorganized.  The files below are the unprocessed types and we have maintained links to the original ACS table numbers.

```{r eval=FALSE}
##Load ACS Data File (IT IS BIG 2.3GB!!)
#Avilable on openICPSR http://doi.org/10.3886/E41333V1
usa.trt <- read.table("R10494789_SL140.csv", header=TRUE, skip=1, sep=",", nrows = 5000)
classes <- sapply(usa.trt, class)
classes[grep("Geo_", names(classes))] <- "factor"
usa.trt <- read.table("R10494789_SL140.csv", header=TRUE, skip=1, sep=",", colClasses = classes)


##REMOVE COLUMNS COLUMNS CONTAINING STANDARD ERRORS
se.col <- grep("*[0-9]s", names(usa.trt)) 
usa.trt <- usa.trt[,-se.col]
```

The table `R10494789_SL140.csv` contains many ACS variables and their standard errors for all census tracts in the US.  We have processed this file such that in a seperate file called `usa_trt_varnames.csv` contains a list describing each column in `R10494789_SL140.csv`.  The table `usa_trt_varnames.csv` includes columns to assist in variable selection and plotting.  The large file is refined by selecting a subset of columns, the selected columns are marked with a $0$ in the `new_set` column.  Replacing a blank or a $1$ in the `new_set` column with a $0$ will include the variable in the analysis, removing a $0$, ore replacing it with a $1$, will remove the associated variable.

```{r}
##Import the manually refined list of variables for inclusion in the classification.
##The column "new_set" indicates if a variable is included in the published analysis.
##The column "desc" is a human readable description of each variable.

#Avilable on openICPSR http://doi.org/10.3886/E41383V1
vars <- read.csv("usa_trt_varnames.csv")

##SELECTED VARIABLES
in.vars <- as.character(vars[vars$new_set==0 & is.na(vars$new_set) == FALSE, "X"])

#Subset the large files to include only the variables indicated in "new_set"
usa.trt <- usa.trt[,in.vars]

##APPLY DESCRIPTIVE NAMES TO DATA
name.vars <- as.character(vars[vars$X %in% in.vars, "desc"])
names(usa.trt) <- name.vars
```

After working with the initial dataset for some time we dscided to include the group quarters population and the population density.  The lines below append variables to `usa.trt`.

```{r eval=FALSE}
#Avilable on openICPSR http://doi.org/10.3886/E41374V1
d.gq <- read.csv("DENSITY_GQ.csv")
usa.trt <- merge(x=usa.trt, y=d.gq, by.y="Geo_GEOID", by.x="Geo_GEOID", all.x=TRUE)
usa.trt <- usa.trt[,-138] #remove geo_id
rm(d.gq)
```

## Splitting the File Into Complete and Incomplete Records

The implementation of $k$-means in R doesn't accept observations with missing data.  We subset the file into complete and incomplete records below

```{r}
#EXTRACT COMPLETE CASES
usa.trt.cc <- usa.trt[complete.cases(usa.trt), ]

#INCOMPELTE CASES
usa.trt.ic <- usa.trt[!complete.cases(usa.trt), ]
```

## MultiCollinearity Check
The block below checks for multicillinearity by regressing each variable on all of the oterh variables.  We report the results in the article.  

```{r}
# Regress all of the variables against each of the input variables.
# The resulting r-squared values saved.
# Variables which have a perfect dependency among them are omitted from the regression. 
# An example of dependency would be something like percent owner occupied and percent renter occupied 
# which together will always 100%

d.fit <- numeric()
for(i in names(usa.trt[,-c(1:4, 10:55, 62:66, 71:81, 89:104, 111:116, 140:144, 121:138)])){
  print(i)
  d <-lm(usa.trt[,as.character(i)] ~., 
         data=usa.trt[,-c(1:4, 10:55, 62:66, 71:81, 89:104, 111:116, 140:144, 121:138,
                             which(x=names(usa.trt)==i, arr.ind=TRUE))])
  d.fit <- append(d.fit, summary(d)$adj.r.squared)
  #break
}
d.fit <- as.numeric(d.fit)

#save the r-squared of the regressions in a data frame
df <- data.frame(var=names(usa.trt[,-c(1:4, 10:55, 62:66, 71:81, 89:104, 111:116, 140:144, 121:138)]), rsq=d.fit)

head(df[order(-df$rsq), ]) #5 highest r-square
tail(df[order(-df$rsq), ]) #5 lowest r-square
```


## Cluster Analysis
This section contains the cluster analysis used to produce the hierarchical classifcation described in the paper.  First we produce 250 clusters via $k$-means then we group these 250 clusters using Ward's Algorithm, an agglomerative hierarchical clustering algorithm.

The first step is to standardize the data.

```{r eval=FALSE}
##STANDARDIZE DATA TO A 0-1 RANGE
range01 <- function(x){(x-min(x))/(max(x)-min(x))}
aa<- apply(USA2[complete.cases(USA2),6:141], MARGIN=2, FUN=range01)

clusters <- list()
fit <- NA
for (i in 1:100000){
  print(paste("starting run", i, sep=" "))
  class.250 <- kmeans(x=aa, centers=250, iter.max=1000000, nstart=1)
  fit[i] <- class.250$tot.withinss
  if (fit[i] < min(fit[1:(i-1)])){
    clusters <- class.250}
}
final <- clusters
```

Then we create a distance matric describing the disimilarity among the 250 clusters.  There are some fun ways to [visualize these matrices](http://stat.ethz.ch/R-manual/R-patched/library/stats/html/heatmap.html).

```{r eval=FALSE}
#distance matrix for cluster centroids.
diss.ctr <- dist(final$centers)
```

Match the incomplete cases to cluster centroids on the basis of fit (i.e. distance).  First we compute the distance between each observation and each cluster centroid, then we assign each case to the minimum distance centroid.  This code block takes awhile to run.
```{r eval=FALSE}
##MATCH INCOMPLETE OBSERVATIONS
dist.k <- data.frame()
for (row in 1:dim(usa.trt.ic)[1]){
  for (k in 1:250){
    dist.k[row, k] <- dist(rbind(final$centers[k,], usa.trt.ic[row, 4:99])) 
  }
}

#Cluster membership for incomplete cases
#select column index for each observation
#column index also represents the cluster id.
mins <- data.frame(cl=NA)
for (row in 1:dim(dist.k)[1]){
  mins[row,] <- ifelse(test=is.na(dist.k[row,]), yes=NA, no=which.min(dist.k[row,]))
}
```

Next some housekeeping, we put everything together into a final data frame and delete large data.frames that are no longer necessary.

```{r eval=FALSE}
##Build final data frame
usa.trt.cc$cluster <- final$cluster
usa.trt.ic$cluster <- mins$cl 
usa.trt.cl <- rbind(usa.trt.cc, usa.trt.ic)
usa.trt.cl$cluster <- as.factor(usa.trt.cl$cluster)                        

##CLEANUP
rm(usa.trt, usa.trt.cc, usa.trt.ic, se.col, mins, dist.k)
```

Ward's method applied to cluster centroids.  Calculate and plot silhouette.

```{r}
wards.ctr <-hclust(diss.ctr, method="ward")

#Silhouette
sil <- NA  #hold fit statistics in a data.frame
for (i in 2:250){
  sil[i-1] <-  summary(silhouette(cutree(wards.ctr,k=i), diss.ctr))$avg.width
}
sil <- data.frame(sil=sil, k=2:249)
```

Plot a dendogram with colorad branches

```{r}
ggplot(data=sil, aes(x=log(k), y=sil, label=k)) +geom_line() + geom_vline(xintercept=log(c(2,10,31,55)), lty=3, lwd=.5)

##COLOR BRANCHES OF DENDOGRAM BY CLASS
clr <- c("#69D2E7","#E0E4CC","#B5B5B5","#79BD9A","#F38630","#EDC951","#8C8590","#CDB380","#547980","#4ECDC4")
grp_names <- c("A: Hispanic and Kids",
               "B: Wealthy Nuclear Families",
               "C: Middle Income, Single Family Homes",
               "D: Native American",
               "E: Wealthy Urbanites",
               "F: Low Income and Diverse",
               "G: Old, Wealthy White",
               "H: Low Income Minority Mix",
               "I: Poor, African-American",
               "J: Residential Institutions, Young People")

# load code of A2R function
source("http://addictedtor.free.fr/packages/A2R/lastVersion/R/code.R")
# colored dendrogram
pdf("ward_clus.pdf")
A2Rplot(wards.ctr, k=10, boxes = FALSE,
        col.up = "gray50", col.down = clr, main="", show.labels = FALSE)
legend("topright", fill=clr, col=clr, 
       legend=grp_names, bg="white", border=NA, box.lwd=0, cex=.5, xjust = 1, yjust = 1)
#text(x=.90 , y=seq(.95,.5, -.05), labels=grp_names, col=clr, cex=.5, pos=2)
dev.off()
```

## Prepare final data frame
The final data frame contains the results of the k-means and Wards, also create a shapefile with results.  The shapefile with results is posted on OpenICPSR.

```{r}
ward.cuts <- data.frame(class=1:250, cutree(wards.ctr,k=c(2,10,31,55)))
usa.trt.cl <- merge(x=usa.trt.cl, y=ward.cuts[, c("X2", "class")], by.x="cluster", by.y="class", all.x=TRUE)
usa.trt.cl <- merge(x=usa.trt.cl, y=ward.cuts[, c("X10", "class")], by.x="cluster", by.y="class", all.x=TRUE)
usa.trt.cl <- merge(x=usa.trt.cl, y=ward.cuts[, c("X31", "class")], by.x="cluster", by.y="class", all.x=TRUE)
usa.trt.cl <- merge(x=usa.trt.cl, y=ward.cuts[, c("X55", "class")], by.x="cluster", by.y="class", all.x=TRUE)

rm(ward.cuts) #cleanup

#load map of census tracts
usa.trt.map  <- readShapePoly("US_tract_2010.shp") #all tracts in the US


##standardize Geo_ID's
usa.trt.cl[nchar(as.character(usa.trt.cl$"FIPS")) < 11,"GEOID2"] <- 
  paste("0", usa.trt.cl[nchar(as.character(usa.trt.cl$"FIPS")) < 11, "FIPS"], sep="")

usa.trt.cl[is.na(usa.trt.cl$GEOID2),"GEOID2"] <- 
  usa.trt.cl[is.na(usa.trt.cl$GEOID2), "FIPS"]

usa.trt.map@data <- data.frame(usa.trt.map@data, 
                               usa.trt.cl[match(usa.trt.map@data[,"GEOID10"],
                                                usa.trt.cl[,"GEOID2"]),])

##LOAD MSA IDs
msas <- read.csv("2010tract_to_dec2009msa.csv", header=FALSE)
msas$GEOID2 <- as.factor(substr(msas$V1,2,12))
names(msas) <- c("id", "msa_id", "msa_name", "msd_id", "msd_name", "GEOID2")

usa.trt.map@data <- data.frame(usa.trt.map@data, 
                               msas[match(usa.trt.map@data[,"GEOID2"], msas[,"GEOID2"]),])


writePolyShape(x=usa.trt.map, fn="US_tract_clusters.shp")
```


## Visual Summaries of Cluster Attributes.
The code below plots heatmaps and other visual zummaries of the clusters.  Plots are grouped by domain, see `vars` or the article for a summary of the variables by domain.

```{r}
##############################
##CREATE SUBSETS AROUND THEMES
##############################
plot_groups <- vars[vars$new_set==0 & is.na(vars$new_set) == FALSE, c("desc", "cat")]
cluster.cols <- c( "X2", "X10", "X31", "X55", "cluster")
##REFINE VARIABLE GROUPS FOR PLOTS
categories <- list(
  EDUCATION = plot_groups[plot_groups$cat %in% c("EDUCATION"), "desc"],
  EMPLOYMENT = plot_groups[plot_groups$cat %in% c("EMPLOYMENT"), "desc"],
  LANGUAGE = plot_groups[plot_groups$cat %in% c("LANGUAGE", "NATIVITY", "RACE"), "desc"],
  WEALTH = plot_groups[plot_groups$cat %in% c("WEALTH"), "desc"],
  HOUSING =  plot_groups[plot_groups$cat %in% c("HOUSING-WEALTH", "HOUSING"), "desc"],
  HOUSING_COST =  plot_groups[plot_groups$cat %in% c("HOUSING-WEALTH"), "desc"],
  MOBILITY = plot_groups[plot_groups$cat %in% c("MOBILITY"), "desc"],
  FAMILY_STABILITY = plot_groups[plot_groups$cat %in% c("FAMILY-STRUCT", "STABILITY"), "desc"])

#NAMES FOR THE 10 CLASS SOLUTION
grp_names <- c("A: Hispanic and Kids",
               "B: Wealthy Nuclear Families",
               "C: Middle Income, Single Family Homes",
               "D: Native American",
               "E: Wealthy Urbanites",
               "F: Low Income and Diverse",
               "G: Old, Wealthy White",
               "H: Low Income Minority Mix",
               "I: Poor, African-American",
               "J: Residential Institutions, Young People")

#################################
##CLUSTER SUMMARIES
#################################
##STANDARDIZE TO FACILITATE DISPLAY
cluster.cols <- c("X10")
for (grp in categories){
  for (cl in cluster.cols){
    if (sum(grp %in% names(usa.trt.cl)) != length(grp)) {
      grp <- grp[grp %in% names(usa.trt.cl)]}
    dataM <- melt(
      data.frame(scale(usa.trt.cl[complete.cases(usa.trt.cl),as.character(grp)]), 
                 cat=usa.trt.cl[complete.cases(usa.trt.cl), as.character(cl)]), id="cat")
    plot1 <- ggplot(dataM, aes(group=variable, x=variable, y=value, fill=variable)) + 
      stat_summary(fun.y = "mean", geom="bar") + facet_grid(.~ cat, scales="free")  + opts(
        axis.text.x = theme_text(size=6, angle=45, hjust=1, vjust=1),
        title="Open Geodemographic \nCommunity Types") + 
      ylab("Variable Mean (Standard Score)") + 
      xlab("Variables by Community Type") 
    print(plot1)
  }}


#####################################
##HEATMAPS OF INDEX SCORES BY DOMAIN
#####################################
##LOAD DATA
#load("tract_data_with_classes_063013.Rdata")
variables <- read.csv("usa_trt_varnames.csv")

##CALCULATE INDEX SCORES
usa.trt.cl$X10 <- factor(x=usa.trt.cl$X10, levels=1:10, labels=LETTERS[seq( from = 1, to = 10 )])
usa.trt.indexScores <- 
  as.data.frame(apply(usa.trt.cl[,5:140], MARGIN=2, FUN=function(x)  (x/mean(x, na.rm=TRUE)) * 100))
usa.trt.indexScores <- cbind(usa.trt.cl[,c(2:4, 141:144)], usa.trt.indexScores)

#tab_summary <- aggregate(.~ X10, data=usa.trt.indexScores[,c(4,8:143)], FUN=summary)
tab_mean  <- aggregate(.~ X10, data=usa.trt.indexScores[,c(4,8:143)], FUN=mean)

##HEATMAPS BY DOMAIN
for (d in levels(variables$Domain)[2:11]){
  vars <- variables[variables$new_set==0 & variables$Domain==d, c("desc", "plotlab")]
  print(paste("DomainSummary", d, ".pdf" ,sep=""))
  assign(d, usa.trt.indexScores[,c("X10", as.character(vars$desc))])
  #eval(substitute(names(x)<-c("X10", as.character(vars$plotlab)), list(x=as.symbol(d))))
  assign(d, aggregate(.~ X10, data=get(d), FUN=mean))
  assign(d, data.frame(X10=get(d)[,1], round(get(d)[,2:dim(get(d))[2]])))
  assign(paste(d, ".m", sep=""), melt(get(d), id="X10"))
  p <- ggplot(get(paste(d, ".m", sep="")), aes(variable, X10, label=value)) +  
    geom_tile(aes(fill = value)) + geom_text(size=3) +
    scale_fill_gradientn(colours=c("#D7191C","#D7191C", "#FDAE61", "#F5F5F5", "#A6D96A", "#1A9641", "#1A9641"),
                         values=c(0, 20, 60, 90, 110, 150, 200, max(get(paste(d, ".m", sep=""))$value)), 
                         breaks=c(0, 50, 100, 150, 200, max(get(paste(d, ".m", sep=""))$value)), 
                         rescaler = function(x, ...) x, oob = identity, name="Index\nScore") +
    scale_y_discrete(expand=c(0,0)) +
    scale_x_discrete(expand=c(0,0), labels=as.character(vars$plotlab)) +
    theme(axis.text.x = element_text(angle = 90, hjust = .5, size=6),
          axis.text.y = element_text(size=6),
          axis.title = element_blank()) + 
    #ylab("Class") + xlab("Measure") +
    #ggtitle(paste(d, "Measures by Class")) + 
    theme(legend.position = "none")
  #ggsave(p, filename=paste("DomainSummary", gsub(pattern=" ", replacement="_", x=d), ".pdf" ,sep=""), height=5, width=(dim(get(d))[2]/2))
}
rm(Demography, Education, "Family Structure", Housing, "Industry of Occupation", Language, Mobility, Race, Stability, Wealth,
   Demography.m, Education.m, "Family Structure.m", Housing.m, "Industry of Occupation.m", Language.m, Mobility.m, Race.m, Stability.m, Wealth.m, vars)

#print a html table of means by class
#print(xtable(tab_mean), type="html", file="~/table.html")

##############################
##POPULATION PYRAMIDS
##############################

##EXTRACT AGE RELATED VARIABLES
plot_vars<-levels(variables$Profile_Plots)[2:17]

age <- usa.trt.cl[,c("X10", as.character(variables[variables$Profile_Plots == "Age_Structure", "desc"]))]

##Merge age categories to make equal range
age$PCT_Male_15_To_19_Years <- age$PCT_Male_18_And_19_Years  +  age$PCT_Male_15_To_17_Years  
age$PCT_Male_20_To_24_Years <- age$PCT_Male_20_Years  + age$PCT_Male_21_Years  + age$PCT_Male_22_To_24_Years   
age$PCT_Male_60_To_64_Years <- age$PCT_Male_60_And_61_Years    +  age$PCT_Male_62_To_64_Years   
age$PCT_Male_65_To_69_Years <- age$PCT_Male_65_And_66_Years  + age$PCT_Male_67_To_69_Years 

age$PCT_Female_15_To_19_Years <- age$PCT_Female_18_And_19_Years  +  age$PCT_Female_15_To_17_Years  
age$PCT_Female_20_To_24_Years <- age$PCT_Female_20_Years  + age$PCT_Female_21_Years  + age$PCT_Female_22_To_24_Years   
age$PCT_Female_60_To_64_Years <- age$PCT_Female_60_And_61_Years    +  age$PCT_Female_62_To_64_Years   
age$PCT_Female_65_To_69_Years <- age$PCT_Female_65_And_66_Years  + age$PCT_Female_67_To_69_Years 

age <- age[,c(1,2,3,4,48,49,10:16,50,51,21:24,25,26,27,52,53,33:39,54,55,44:47)]

age_m <- melt(age, id.vars="X10")
#age_m$variable <- as.factor(age_m$variable, ordered=TRUE)

##SUMMARIZE VARIABLES BY GROUP
age_m <- aggregate(value~variable + X10, data=age_m, FUN=summary, na.rm=TRUE)

##ATTRIBUTE GENDER TO EARCH ROW FOR SUBSETTING
age_m$gender <- ifelse(
  age_m$variable %in% 
    names(age)[2:19],
  "Male", "Female")

##ADD MEAN ANF QUARTILES TO SUMMARY TABLE
age_m$Mean <- age_m$value[,4]
age_m$Q1 <- age_m$value[,2]
age_m$Q3 <- age_m$value[,5]

##REMOVE "SUMMARY" MATRIX FROM DATA FRAME
age_m <- age_m[,c(1,2,4,5,6,7)]

##STANDARDIZE VARIABLE NAMES FOR MALE AND FEMALE
##PRESERVE ORDER OF VARIABLE
age_mm <- age_m[age_m$gender=="Male", ]
age_mf <- age_m[age_m$gender=="Female", ]
age_mm$variable <- factor(age_mm$variable, ordered=TRUE) 
age_mf$variable <- factor(age_mf$variable, ordered=TRUE) 
levels(age_mm$variable) <- substr(levels(age_mm$variable), start=10, stop=nchar(as.character(levels(age_mm$variable))))
levels(age_mf$variable) <- substr(levels(age_mf$variable), start=12, stop=nchar(as.character(levels(age_mf$variable))))

##MAKE ALL FEMALE DATA NEGATIVE FOR PLOTTING
age_mf$Mean <-  age_mf$Mean * -1
age_mf$Q1 <-  age_mf$Q1 * -1
age_mf$Q3 <-  age_mf$Q3 * -1

###PLOT POPULATION PYRAMIDS
ggplot(data=age_mm,
       aes(y=Mean, x=variable, ymin=Q1, ymax=Q3)) + 
  geom_pointrange() + facet_grid(X10~.) +
  geom_pointrange(data=age_mf,
                  aes(y=Mean, x=variable, ymin=Q1, ymax=Q3)) +
  theme(axis.text.y = element_blank(),
        axis.text.x = element_blank(),
        axis.title = element_blank(),
        panel.grid.minor.y=element_blank(), panel.grid.major.y=element_blank()) +
  coord_flip()

#cleanup
rm(age,age_m, age_mm, age_mf)

########################################
##HEATMAPS FOR TYPES (SUBCLASSES OF X10)
########################################
##HEATMAPS BY DOMAIN
##FOR SUBCLASS K
#K <- 4 #GROUP NUMBER
#subclasses <-  2 #number of subclasses for class K
##FOR EXAMPLE GROUP 2 HAS 11 SUBCLASSES (TYPES)
K <- 2 
subclasses <-  11 #number of subclasses for class K
for (d in levels(variables$Domain)[2:11]){
  vars <- variables[variables$new_set==0 & variables$Domain==d, c("desc", "plotlab")]
  print(paste("DomainSummary", d, ".pdf" ,sep=""))
  assign(d, usa.trt.indexScores[usa.trt.indexScores$X10 == K,c("X55", as.character(vars$desc))])
  assign(d, 
         aggregate(.~ X55, data=get(d), FUN=mean))
  assign(paste(d, ".m", sep=""), melt(round(get(d)), id="X55"))
  p <- ggplot(get(paste(d, ".m", sep="")), aes(x=variable, y=as.character(X55), label=value)) +  
    geom_tile(aes(fill = value)) + geom_text(size=3) +
    scale_fill_gradientn(colours=c("#D7191C","#D7191C", "#FDAE61", "#F5F5F5", "#A6D96A", "#1A9641", "#1A9641"),
                         values=c(0, 20, 60, 90, 110, 150, 200, max(get(paste(d, ".m", sep=""))$value)), 
                         breaks=c(0, 50, 100, 150, 200, max(get(paste(d, ".m", sep=""))$value)), 
                         rescaler = function(x, ...) x, oob = identity, name="Index\nScore") +
    scale_y_discrete(expand=c(0,0)) +
    scale_x_discrete(expand=c(0,0), labels=as.character(vars$plotlab)) +
    theme(axis.text.x = element_text(angle = 90, hjust = .5),
          axis.title = element_blank()) + 
    #ylab("Class") + xlab("Measure") +
    #ggtitle(paste(d, "Measures by Class")) + 
    theme(legend.position = "none")
  ggsave(p, filename=paste("DomainSummary55class", K, gsub(pattern=" ", replacement="_", x=d), ".pdf" ,sep=""), height=5, width=(dim(get(d))[2]/2))
}
```
