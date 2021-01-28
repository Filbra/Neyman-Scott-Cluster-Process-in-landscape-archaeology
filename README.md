# Archaeo_Neyman_Scott_Process
---
This R script code was developed by dr. F. Brandolini (Newcastle University, UK) to accompany the paper: *Costanzo S., Brandolini F., Ahmed H., Zerboni A., Manzo A - Creating the funerary landscape in eastern Sudan*, submitted to Scientific Reports, 20XX.
author: "Filippo Brandolini"
date: "26/01/2021"
---

## Load Libraries
```{r echo=FALSE, results='hide', include=FALSE}

LibPack<-c("raster","spatstat","rgdal","rgeos","maptools","RColorBrewer","sp","MASS","corrplot","ggplot2","ggforce","ggspatial","gridExtra")

lapply(LibPack, require, character.only=TRUE)
```
The following libraries were loaded: 'LibPack'.

# Introduction

Funeral landscapes are a specific example of archaeological landscapes that embed the relationship between environment, burials and human behaviour. In this study, we employ the Point Pattern Analysis (PPA) technique to explore the choice of location of a desk-created dataset of several thousands stone-built medieval Islamic funerary monuments (qubbas), found along the foothills of isolated rocky outcrops in the remote semi-arid Kassala province of Eastern Sudan. In the region a comprehensive geomorphological and geoarchaeological survey was carried out in the field to assess the main archaeological features, highlighting the existence of an impressive number of archaic and modern qubbas, whose spatial distribution can be interpreted using specific tools, including, for the first time in archaeology, of the **Neyman-Scott Cluster Process**.

Two complementary null hypotheses were tested through Point Pattern Analysis (PPA):

*H1*: At landscape-scale, the density of qubbas is uniform (the intensity of the point pattern is stationary and isotropic). The spatial variables do not affect the spatial distribution of the points.

*H2*: At local-scale, the qubbas are completely independent from each other presenting no spatial correlation 

# Part 1: Import Data 
The geomorphological analysis of the region served as a starting point for determining a list of potentially explanatory variables. The Elevation values were retrieved from the ALOS World 3D, a global 30-meters resolution  Digital Surface Model (DSM). The DSM was then elaborated in GRASS GIS and in SAGA GIS to create the spatial variables used in the PPA. The desk-created dataset comprises 783 tumuli and 10274 qubbas.

```{r results='hide'}

# Topographic variables
dem <- raster("GeoTiff/Elevation.tif")
slp <- raster("GeoTiff/Slope.tif")
asp <- raster("GeoTiff/Aspect.tif")

# Aspect Northness (ns) & Eastness (es) [rad]
writeRaster((cos(asp)* pi / 180),"GeoTiff/Northness.tif", format="GTiff",
            overwrite=TRUE)
ns <- raster("GeoTiff/Northness.tif")

writeRaster((sin(asp)* pi / 180),"GeoTiff/Eastness.tif", format="GTiff",
            overwrite=TRUE)
es <- raster("GeoTiff/Eastness.tif")

# Topographic Position Index (TPI)
TPI_100<- raster("GeoTiff/TPI100.tif")
TPI_500<- raster("GeoTiff/TPI500.tif")
TPI_1000<- raster("GeoTiff/TPI1000.tif")

# Convergence Index (CI)
CI <- raster("GeoTiff/CI.tif")

# Geomorphon
geomorph <- raster("GeoTiff/Geomorphons.tif")

# Euclidean distances from resources
igne_eu <- raster("GeoTiff/I.R.Dist.tif")
meta_eu <- raster("GeoTiff/M.R.Dist.tif")
stream_eu <-raster("GeoTiff/S.dist.tif")

# Tumuli's Cumulative Viewshed Analysis (CVA)
cva <-raster("GeoTiff/CVA.tif")

# Qubbas sites
sites_q <- remove.duplicates(readOGR(dsn="EsriSHP/qubbas", layer="qubbas"))

# Tumuli sites
sites_t <- remove.duplicates(readOGR(dsn="EsriSHP/tumuli", layer="tumuli"))

# Region Of Interest (ROI)
prj_area <- readOGR(dsn="EsriSHP/ROI", layer="ROI")

# Setting the working region
region<-as(as(prj_area,"SpatialPolygons"),"owin")

# Converting qubbas site locations to point pattern process (ppp)
q_ppp<- as.ppp(coordinates(sites_q), region)

# Save files loaded into R ####

save(dem, TPI_100,TPI_500, TPI_1000, CI, slp, ns, es,igne_eu, meta_eu, stream_eu,cva,
     sites_q, sites_t, prj_area, file="PPA_Data.RData")

# Number of Simulations
NuSim <- 999

```

# Part 2: Non-parametric analysis: Kernel Density Estimation (KDE)

The intensity of sites was initially explored through the KDE that was obtained with the spatstat function *density.ppp*. The smoothing bandwidth can be specified in the function with the argument *sigma*. The kernel bandwidth sigma controls the degree of smoothing: a small value may omit a potential general trend, while a large value may omit local details. In this research, the sigma was automatically estimated with the likelihood cross-validation method in *spatstat* (*bw.ppl*) and manually adjusted through the function argument adjust (Fig.1).

```{r results='hide'} 
# Rescaling q_ppp from m to Km (better plot representation !)
q_ppp_KM<- rescale((as.ppp(coordinates(sites_q), region)), 1000, "km")

# Automatic BE 
ppl = bw.ppl(q_ppp_KM)

# Adjusting BE manually
MBE1 <- density.ppp(q_ppp_KM)
MBE2 <- density.ppp(q_ppp_KM, sigma = ppl)
MBE3 <- density.ppp(q_ppp_KM, sigma = ppl, adjust= 10)
MBE4 <- density.ppp(q_ppp_KM, sigma = ppl, adjust= 20)
MBE5 <- density.ppp(q_ppp_KM, sigma = ppl, adjust= 30)
MBE6 <- density.ppp(q_ppp_KM, sigma = ppl, adjust= 40)
```

```{r echo=FALSE, results='hide'}
#Plotting the (Fig.1) Density maps - Bandwidth Estimation with GGPLOT2

K1<- fortify(data.frame(MBE1))
K2<- fortify(data.frame(MBE2))
K3<- fortify(data.frame(MBE3))
K4<- fortify(data.frame(MBE4))
K5<- fortify(data.frame(MBE5))
K6<- fortify(data.frame(MBE6))

sitesdf<-fortify(data.frame(sites_q), region = prj_area)

png(width=2500,height=1500, res = 200, file="Output/Bandwidth Estimation.png")

KD1<-ggplotGrob(ggplot(K1, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
                             shape= 1, size= 0.1, colour= "Black", bg="Black")+
                  labs(title = "Defaut Sigma", adj= 1, x = "",y = "")+
                  coord_equal())
KD2<-ggplotGrob(ggplot(K2, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  labs(title = "Automatic Sigma (ppl)", x = "",y = "")+
                  coord_fixed())
KD3<-ggplotGrob(ggplot(K3, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
                             shape= 1, size= 0.1, colour= "Black", bg="Black")+
                  labs(title = "Automatic Sigma (ppl) adjusted: 10", x = "",y = "")+
                  coord_fixed())
KD4<-ggplotGrob(ggplot(K4, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
                             shape= 1, size= 0.1, colour= "Black", bg="Black")+
                  labs(title = "Automatic Sigma (ppl) adjusted: 20", x = "",y = "")+
                  coord_fixed())
KD5<-ggplotGrob(ggplot(K5, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
                             shape= 1, size= 0.1, colour= "Black", bg="Black")+
                  labs(title = "Automatic Sigma (ppl) adjusted: 30", x = "",y = "")+
                  coord_fixed())
KD6<-ggplotGrob(ggplot(K6, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                  scale_fill_distiller(type = "seq",palette = "Spectral", 
                                       guide = "colourbar", aesthetics = "fill")+
                  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
                             shape= 1, size= 0.1, colour= "Black", bg="Black")+
                  labs(title = "Automatic Sigma (ppl) adjusted: 40", x = "",y = "")+
                  coord_fixed())
grid.arrange(KD1,KD2,KD3,KD4,KD5,KD6,
             widths = c(1,1,1),
             layout_matrix = rbind(c(1, 2, 3),
                                   c(4, 5, 6)))
dev.off()

```

The best representation of the dynamic intensity of the qubbas in the ROI’s landscape was obtained by a manual adjustment of the *sigma* at *adjust = 30* (Fig.2).

```{r echo=FALSE, results='hide'}
##Plotting (Fig.2): KDE of the study area with GGPLOT2

KDE <- fortify(data.frame(MBE5))

png(width=1000,height=1200,res= 200, file="Output/KDE.png")

KDE_map<-ggplot(KDE, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
  scale_fill_distiller(type = "seq",palette = "Spectral", 
                       guide = "colourbar", aesthetics = "fill")+
  geom_point(data = sitesdf, aes(x=coords.x1/1000, y=coords.x2/1000), 
             shape= 1, size= 0.1, colour= "Black", bg="Black")+
  labs(title = "Kernel Density Analysis Sigma via ppl (adjusted)", adj= 1, x = "",y = "")+
  coord_equal()
print(KDE_map)
dev.off()
```

# Spatial Covariates

To avoid overparameterization in the models, the Pearson correlation’s test was employed to test collinearity between the variables. Collinearity is the high linear relation of two or more predictors. In general, an absolute correlation coefficient of >0.7 (dark blue or dark red) among two or more predictors indicates the presence of collinearity. In this research we considered > 0.6 as a parameter to exclude correlated variables (Fig.3).

## Testing Collinearity between covariates

```{r results='hide'} 
RandomC<- as.data.frame((sampleRandom(
  (stack(dem, TPI_100,TPI_500, TPI_1000, CI, slp, ns, es, 
         geomorph,igne_eu, meta_eu, stream_eu, cva)), 999)))

coll_test <- cor(RandomC, method = "pearson")
```

```{r echo=FALSE, results='hide'}
#Plotting (Fig.3): the correlogram with GGPLOT2

png(width=1500,height=1500,res= 200, file="Output/Pearson's correlation test results.png")
corrplot(coll_test, type="upper", method = "color", order="hclust",tl.cex= 0.7, tl.col = "black", tl.srt = 90,
         addCoef.col = "black", number.font=1, number.cex= 0.6, diag = FALSE)
dev.off()
```
The topographic and morphometric variables that showed less collinearity (Slope, Northness, Eastness, TPI100, CI and Geomorphon, distance from metamorphic rocks and CVA) were furtherly investigated to evaluate their actual influence on the point process (Fig. 4).

```{r echo=FALSE, results='hide', warning=FALSE}
# Histogram Sites vs variable

coord_sites <- coordinates(sites_q)

# Sites vs Slope

Sites_Slope<- cbind(coord_sites,
                    (extract(slp, coord_sites, fun=TRUE)))
colnames(Sites_Slope)<-c("X","Y","Sl")
SSlDf <-data.frame(Sites_Slope)

# Sites vs Northness

Sites_N<- cbind(coord_sites,
                (extract(ns, coord_sites, fun=TRUE)))
colnames(Sites_N)<-c("X","Y","N")
SNDf <-data.frame(Sites_N)

# Sites vs Eastness

Sites_E<- cbind(coord_sites,
                (extract(es, coord_sites, fun=TRUE)))
colnames(Sites_E)<-c("X","Y","E")
SEDf <-data.frame(Sites_E)

# Sites vs TPI 100

Sites_Tpi <- cbind(coord_sites,
                   (extract(TPI_100, coord_sites, fun=TRUE)))
colnames(Sites_Tpi)<-c("X","Y","Tp")
SSTpDf <-data.frame(Sites_Tpi)

# Sites vs TCI

Sites_Ci <- cbind(coord_sites,
                  (extract(CI, coord_sites, fun=TRUE)))
colnames(Sites_Ci)<-c("X","Y","Ci")
SSTcDf <-data.frame(Sites_Ci)

# Sites vs Geomorphon

Sites_Geo <- cbind(coord_sites,
                   (extract(geomorph, coord_sites, fun=TRUE)))
colnames(Sites_Geo)<-c("X","Y","G")
SSGDf <-data.frame(Sites_Geo)

# Sites vs Metamorphic Rocks

sites_teta <- cbind(coord_sites,
                    (extract(meta_eu, coord_sites, fun=TRUE)))
colnames(sites_teta)<-c("X","Y","M")
SSMDf <-data.frame(sites_teta)

# Sites vs CVA

Sites_CVA <- cbind(coord_sites,
                   (extract(cva, coord_sites, fun=TRUE)))
colnames(Sites_CVA)<-c("X","Y","Cv")
SCVADf <-data.frame(Sites_CVA)

# Plot (Fig. 4)with ggplotGrob & GGPLOT2

png(width=5000,height=2500, res = 200, file="Output/Histograms Sites vs Variables.png")

SSl<-ggplotGrob (ggplot(data= SSlDf, aes(x=Sl))+
                   geom_histogram(aes(y=stat(density)),
                                  binwidth=2.5, 
                                  fill="light yellow",
                                  color="black",
                                  alpha= 0.4)+
                   geom_density(size=1, col="Red")+
                   scale_x_continuous(breaks = seq(0,30,2.5), lim = c(0,30))+
                   scale_y_continuous(labels = scales::percent_format())+
                   labs(title = "Sites occurrence according to Slope", adj= 0.5,
                        x = "Slope (°)",y = "Density")+
                   theme(text = element_text(size=10)))

SSN<-ggplotGrob (ggplot(data= SNDf, aes(x=N))+
                   geom_histogram(aes(y=stat(density)),
                                  binwidth=0.005, 
                                  fill="light yellow",
                                  color="black",
                                  alpha= 0.4)+
                   geom_density(size=1, col="Red")+
                   scale_x_continuous(breaks = seq(-0.0175,0.0175,0.005), lim = c(-0.0175,0.0175))+
                   scale_y_continuous(labels = scales::percent_format())+
                   labs(title = "Sites occurrence according to Northness",cex=0.2, adj= 0.5,
                        x = "Northness (rad)",y = "Density")+
                   theme(text = element_text(size=10)))

SSE<-ggplotGrob (ggplot(data= SEDf, aes(x=E))+
                   geom_histogram(aes(y=stat(density)),
                                  binwidth=0.005, 
                                  fill="light yellow",
                                  color="black",
                                  alpha= 0.4)+
                   geom_density(size=1, col="Red")+
                   scale_x_continuous(breaks = seq(-0.0175,0.0175,0.005), lim = c(-0.0175,0.0175))+
                   scale_y_continuous(labels = scales::percent_format())+
                   labs(title = "Sites occurrence according to Eastness", adj= 0.5,
                        x = "Eastness (rad)",y = "Density")+
                   theme(text = element_text(size=10)))


ST100<-ggplotGrob (ggplot(data= SSTpDf, aes(x=Tp))+
                     geom_histogram(aes(y=stat(density)),
                                    binwidth=1, 
                                    fill="light yellow",
                                    color="black",
                                    alpha= 0.4)+
                     geom_density(size=1, col="Red")+
                     scale_x_continuous(breaks = seq(-8,8,1), lim = c(-8,8))+
                     scale_y_continuous(labels = scales::percent_format())+
                     labs(title = "Sites occurrence according to TPI", adj= 0.5,
                          x = "TPI value",y = "Density")+
                     theme(text = element_text(size=10)))

SCI<-ggplotGrob (ggplot(data= SSTcDf, aes(x=Ci))+
                   geom_histogram(aes(y=stat(density)),
                                  binwidth=2, 
                                  fill="light yellow",
                                  color="black",
                                  alpha= 0.4)+
                   geom_density(size=1, col="Red")+
                   scale_x_continuous(breaks = seq(0,50,5), lim = c(0,50))+
                   scale_y_continuous(labels = scales::percent_format())+
                   labs(title = "Sites occurrence according to CI", adj= 0.5,
                        x = "CI value",y = "Density")+
                   theme(text = element_text(size=10)))

SGEO<-ggplotGrob (ggplot(data= SSGDf, aes(x=G))+
                    geom_histogram(aes(y=stat(density)),
                                   binwidth=1, 
                                   fill="light yellow",
                                   color="black",
                                   alpha= 0.4)+
                    geom_density(size=1, col="Red")+
                    scale_x_continuous(breaks = seq(0,12,1), lim = c(0,12))+
                    scale_y_continuous(labels = scales::percent_format())+
                    labs(title = "Sites occurrence according to Geomorphon", adj= 0.5,
                         x = "Geomorphon value",y = "Density")+
                    theme(text = element_text(size=10)))

SM<-ggplotGrob (ggplot(data= SSMDf, aes(x=M))+
                  geom_histogram(aes(y=stat(density)),
                                 binwidth=25,
                                 fill="light yellow",
                                 color="black",
                                 alpha= 0.4)+
                  geom_density(size=1, col="Red")+
                  scale_x_continuous(breaks = seq(0,200,50), lim = c(0,200))+
                  scale_y_continuous(labels = scales::percent_format())+
                  labs(title = "Sites occurrence according to Metamorphic Rocks", adj= 0.5,
                       x = "Distance (m)",y = "Density")+
                  theme(text = element_text(size=10)))

SCVA<-ggplotGrob (ggplot(data= SCVADf, aes(x=Cv))+
                    geom_histogram(aes(y=stat(density)),
                                   binwidth=25,
                                   fill="light yellow",
                                   color="black",
                                   alpha= 0.4)+
                    geom_density(size=1, col="Red")+
                    scale_x_continuous(breaks = seq(0,200,50), lim = c(0,200))+
                    scale_y_continuous(labels = scales::percent_format())+
                    labs(title = "Sites occurrence according to Tumuli's CVA", adj= 0.5,
                         x = "Distance (m)",y = "Density")+
                    theme(text = element_text(size=10)))

grid.arrange(SSl,SSN,SSE,ST100,SCI,SGEO,SM,SCVA,
             widths = c(1,1, 1, 1),
             layout_matrix = rbind(c(1,2,3,4),
                                   c(5, 6,7,8)))

dev.off()
```
Both Northness and Eastness were excluded from the parametric estimation of intensity as they show seem to have no influence on the landscape spatial distribution of the qubbas: the points are uniformly oriented in all cardinal directions. The six resulting covariates employed for the PPM are: Slope, Aspect, Topographic Position Index (TPI 100), Convergence Index (CI), Geomorphon, Euclidean distances from outcrops of metamorphic rock, Tumuli’s  Cumulative Viewshed Analysis (CVA) (Fig. 5). 

```{r echo=FALSE, results='hide', warning=FALSE}

# Plotting (Fig. 5), the selected covariates

Slopedf<- data.frame(rasterToPoints(slp))
colnames(Slopedf) <- c("x","y","value")
head(Slopedf)

TPIdf<- data.frame(rasterToPoints(TPI_100))
colnames(TPIdf) <- c("x","y","value")
head(TPIdf)

CIdf<- data.frame(rasterToPoints(CI))
colnames(CIdf) <- c("x","y","value")
head(CIdf)

Geodf<- data.frame(rasterToPoints(geomorph))
colnames(Geodf) <- c("x","y","value")
head(Geodf)

Metadf<- data.frame(rasterToPoints(meta_eu))
colnames(Metadf) <- c("x","y","value")
head(Metadf)

CVAdf<- data.frame(rasterToPoints(cva))
colnames(CVAdf) <- c("x","y","value")
head(CVAdf)

png(width=4000,height=2000, res = 150, file="Output/Spatial covariates selected.png")

S<-ggplotGrob(ggplot(Slopedf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                scale_fill_distiller(type = "seq",palette = "BrBG", 
                                     guide = "colourbar", aesthetics = "fill")+
                labs(title = "Slope", x = "",y = "")+
                theme_grey(base_size = 15)+
                coord_fixed())
Ti<-ggplotGrob(ggplot(TPIdf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                 scale_fill_distiller(type = "seq",palette = "BrBG", 
                                      guide = "colourbar", aesthetics = "fill")+
                 labs(title = "TPI (100)", x = "",y = "")+
                 theme_grey(base_size = 15)+
                 coord_fixed())
Ci<-ggplotGrob(ggplot(CIdf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                 scale_fill_distiller(type = "seq",palette = "BrBG", 
                                      guide = "colourbar", aesthetics = "fill")+
                 labs(title = "CI", x = "",y = "")+
                 theme_grey(base_size = 15)+
                 coord_fixed())
G<-ggplotGrob(ggplot(Geodf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                scale_fill_distiller(type = "seq",palette = "BrBG", 
                                     guide = "colourbar", aesthetics = "fill")+
                labs(title = "Geomorphons", x = "",y = "")+
                theme_grey(base_size = 15)+
                coord_fixed())
M<-ggplotGrob(ggplot(Metadf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                scale_fill_distiller(type = "seq",palette = "BrBG", 
                                     guide = "colourbar", aesthetics = "fill")+
                labs(title = "Euclidean dist. from Metamorphic Rocks", x = "",y = "")+
                theme_grey(base_size = 15)+
                coord_fixed())
Cv<-ggplotGrob(ggplot(CVAdf, aes(x=x, y=y)) + geom_tile(aes(fill = value))+
                 scale_fill_distiller(type = "seq",palette = "BrBG", 
                                      guide = "colourbar", aesthetics = "fill")+
                 labs(title = "Tumuli's CVA", x = "",y = "")+
                 theme_grey(base_size = 15)+
                 coord_fixed())

grid.arrange(S,Ti,Ci,G,M,Cv,
             widths = c(1,1,1),
             layout_matrix = rbind(c(1,2,3),
                                   c(4,5,6)))

dev.off()
```

# Part 3: Parametric analysis

## First-order properties

With the *spatstat* function *ppm* three different Inhomogeneous Poisson Process (IPP) models have been fitted: Model 1 (Topographic covariates), Model 2 (Lithological covariate), Model 3 (Socio-cultural covariate). 

```{r results='hide', warning=FALSE}

# Convert covariates to objects of class image and create list

covar<-list(Elev=as.im(as(dem,"SpatialGridDataFrame")),
            TPI=as.im(as(TPI_100,"SpatialGridDataFrame")),
            CI=as.im(as(CI,"SpatialGridDataFrame")),
            Geo=as.im(as(geomorph,"SpatialGridDataFrame")),
            Slope=as.im(as(slp,"SpatialGridDataFrame")),
            NS=as.im(as(ns,"SpatialGridDataFrame")),
            ES=as.im(as(es,"SpatialGridDataFrame")),
            Meta=as.im(as(meta_eu,"SpatialGridDataFrame")),
            CVA=as.im(as(cva,"SpatialGridDataFrame")))

# Model 0: homogeneous point process - Null model

q_mod_0<- ppm(q_ppp ~ 1)

# Model 1: parametric inhomogeneous point process - Topographic variables

q_mod_1<- ppm(q_ppp, ~ Slope + TPI + CI + Geo, data = covar)

# Bayesian Information Criterion (BIC): stepwise variable selection

q_mod_1BIC<-stepAIC(q_mod_1,k=log(length(sites_q)))

# Model 2: parametric inhomogeneous point process - Buiding Resource

q_mod_2<- ppm(q_ppp, ~ Meta, data = covar)

# Bayesian Information Criterion (BIC): stepwise variable selection

q_mod_2BIC<-stepAIC(q_mod_2,k=log(length(sites_q)))

# Model 3: parametric inhomogeneous point process - CVA

q_mod_3<- ppm(q_ppp, ~ CVA, data = covar)

# Bayesian Information Criterion (BIC): stepwise variable selection

q_mod_3BIC<-stepAIC(q_mod_3,k=log(length(sites_q)))

# Model 4: parametric inhomogeneous point process - hybrid model

q_mod_4<- ppm(q_ppp, ~ Slope + CI + TPI + Geo + Meta + CVA, data = covar)

# Bayesian Information Criterion (BIC): stepwise variable selection

q_mod_4BIC<-stepAIC(q_mod_4,k=log(length(sites_q)))

# Exporting Tables

# Create PPM lists

ppp_q_models<-list(model0=q_mod_0,model1=q_mod_1BIC,model2=q_mod_2BIC,model3=q_mod_3BIC,model4=q_mod_4BIC)

# Export PPM summary

write.table(capture.output(print(ppp_q_models)),file="Output/PPP_Models.txt")

# Compare BIC scores for the different models

q_BIC<-lapply(ppp_q_models,BIC)
write.table(q_BIC,file="Output/BIC_Models_scores.txt")

# BIC weight for diferent models (Credits: dr. Francesco Carrer, Newcastle University)

AIC.BIC.weight<-function(x){
  for(i in 1:length(x)){
    x.vect<-as.numeric(c(x[1:i]))}
  delta<-x.vect-min(x.vect)
  L<-exp(-0.5*delta)
  result<-round(L/sum(L),digits=7)
  return(result)
}

write.table(AIC.BIC.weight(q_BIC[1:2]),file="Output/BIC_Models01_weights.txt")
write.table(AIC.BIC.weight(q_BIC[1:3]),file="Output/BIC_Models02_weights.txt")
write.table(AIC.BIC.weight(q_BIC[1:4]),file="Output/BIC_Models03_weights.txt")
write.table(AIC.BIC.weight(q_BIC[1:5]),file="Output/BIC_Models04_weights.txt")
```

The Schwarz’s Bayesian Information Criterion (BIC) has been employed to compare the competing models and to assess the performance of each covariate. BIC is calculated as the difference between the maximised likelihood of the model and the product of covariates and number of observations (points), therefore a lower BIC corresponds to a better performing model. Following the principle of parsimony, step-wise selection of covariates enables the identification of the combination of variables that minimises BIC values, and the covariates that show less contribution to the performance of the models are excluded during the process (Tab. 1). The remaining covariates were then employed to fit a fourth “hybrid” model that includes the best performing variables (Model 4).

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-c3ow{border-color:inherit;text-align:center;vertical-align:top}
.tg .tg-7btt{border-color:inherit;font-weight:bold;text-align:center;vertical-align:top}
</style>
<table class="tg">
<caption> Tab. 1 - Result of the BIC stepwise covariates selection and model selection based on BIC weights.</caption>
<thead>
  <tr>
    <th class="tg-7btt">PPMs</th>
    <th class="tg-7btt">Selected Covariates</th>
    <th class="tg-7btt">Discarded<br>Covariates</th>
    <th class="tg-7btt">BIC</th>
    <th class="tg-7btt">Weights<br>Model 0–1</th>
    <th class="tg-7btt">Weights<br>Model 0–2</th>
    <th class="tg-7btt">Weights<br>Model 0–3</th>
    <th class="tg-7btt">Weights<br>Model 0–4</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">285618.727843511</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
  </tr>
  <tr>
    <td class="tg-c3ow">1</td>
    <td class="tg-c3ow">Slope, TPI100, CI, Geomorphon</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">267576.992701805</td>
    <td class="tg-c3ow">1</td>
    <td class="tg-c3ow">1</td>
    <td class="tg-c3ow">1</td>
    <td class="tg-c3ow">0</td>
  </tr>
  <tr>
    <td class="tg-c3ow">2</td>
    <td class="tg-c3ow">Eu. Dist. Metamorphic Rocks</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">279957.068568553</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
  </tr>
  <tr>
    <td class="tg-c3ow">3</td>
    <td class="tg-c3ow">Tumuli’s CVA</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">284537.742730204</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">0</td>
    <td class="tg-c3ow">0</td>
  </tr>
  <tr>
    <td class="tg-c3ow">4</td>
    <td class="tg-c3ow">Slope, TPI100, CI, Geomorphon, Eu. Dist. Metamorphic Rocks, Tumuli’s CVA</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">266045.103883927</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">-</td>
    <td class="tg-c3ow">1</td>
  </tr>
</tbody>
</table>

During the step-wise selection no covariates have been excluded in Model 1, 2, 3 meaning that all the spatial variables show significant correlation with the points. Thus, the hybrid model (Model 4) has been fitted using all the six initially selected covariates and it scored the lowest BIC value (Tab. 1). The coefficients of the hybrid (Model 4) show an inverse correlation of the sites’ locational pattern with Slope, TPI and the distances from metamorphic rocks (Eu. Dist. Meta). Conversely, the coefficients of BIC-selected Model 4 show a direct correlation with CI, Geomorphon and CVA (Tab. 2).

<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-baqh{text-align:center;vertical-align:top}
</style>
<table class="tg">
<caption> Tab. 2 - Covariates of the BIC-selected Model 4.</caption>
<thead>
  <tr>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">Covariates</span></th>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">Estimate</span></th>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">S.E.</span></th>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">CI 95% lo</span></th>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">CI 95% hi</span></th>
    <th class="tg-baqh"><span style="font-weight:700;font-style:normal;text-decoration:none;background-color:transparent">Z test</span></th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">Slope</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-4.492999e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.449379e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-7.333730e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-1.652268e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">CI</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">5.986409e-02</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">6.500641e-04</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">5.858999e-02</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">6.113819e-02</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">TPI</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-1.309407e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">4.892103e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.405291e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-1.213524e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">Geomorphon</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.695567e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">3.900717e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.619115e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.772020e-01</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">Eu. Dist. Meta</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-6.269355e-05</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.828274e-06</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-6.627690e-05</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">-5.911020e-05</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
  <tr>
    <td class="tg-baqh"><span style="font-style:normal;text-decoration:none;color:#000;background-color:transparent">CVA</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.433004e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.617823e-04</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.115916e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#000;background-color:transparent">1.750091e-03</span></td>
    <td class="tg-baqh"><span style="font-weight:400;font-style:normal;text-decoration:none;color:#333;background-color:#FFF">&lt;0.001</span></td>
  </tr>
</tbody>
</table>

## Second-order properties

The interpoint dependence of sites (*H2*) has been assessed applying the **Besag’s L function**. Since spatial interaction tends to occur at small scale, the estimation of L function has been limited to 1.5 km distance (Fig. 6).

```{r results='hide'}
Linhom_0 <- envelope(q_mod_0, fun=Lest,
                     correction="trans", r=seq(0,1500,50), nsim=NuSim)

Linhom_1 <- envelope(q_mod_4, fun=Lest,
                     correction="trans", r=seq(0,1500,50), nsim=NuSim)
```

```{r echo=FALSE, results='hide'}

# Plot (Fig. 6) L functions (mod. 0 & mod 4)

png(width=2000,height=1000,units="px",res=150, file="Output/Results of the L function on model 0 and model 4.png")
par(mar=c(4,4,4,4))
par(mfrow=c(1,2))
plot(Linhom_0,main="L function (Mod_0)",xlab="Distance (m)", ylab="L function (r)",
     legend=F)
legend("topleft",legend=c("Observed","Expected","Confidence Envelope"),lty=c(1,2,NA),
       col=c(1,2,"lightgrey"),lwd=c(1,1,NA),pch=c(NA,NA,15),cex=0.8,pt.cex=1.5,
       y.intersp=1.5)
plot(Linhom_1,main="L function (Mod_4)",xlab="Distance (m)", ylab="L function(r)",
     legend=F)
legend("topleft",legend=c("Observed","Expected","Confidence Envelope"),lty=c(1,2,NA),
       col=c(1,2,"lightgrey"),lwd=c(1,1,NA),pch=c(NA,NA,15),cex=0.8,pt.cex=1.5,
       y.intersp=1.5)
dev.off()
par(mfrow=c(1,1))
```

The observed values completely exceed the 95th percentile of simulated values (the highest limit of the confidence envelope) both in the homogeneous (Model 0) and inhomogeneous (Model 4) plots, suggesting an extremely high positive correlation of points that cannot be explained with landscape preferences alone. Because even the KDE highlighted hot-spots in the intensity of the sites and the nearest neighbour (nn) distances evidence that the qubbas are very close placed (min nn distance: 0,72 m; mean nn distance: 23,3 m), the occurrence of a positive dependence between points was tested fitting a cluster process model.

# Fitting Neyman-Scott Cluster Point Processes (KPPM)

In this research, a **Neyman Scott Cluster process (NSC)** was employed. A variant of the NSC process called *Thomas cluster process* was employed, using the argument *method = Thomas* in the *kppm* function. Then, two different *kppm* have been fitted: a stationary Thomas Cluster model (Model 5) and an inhomogeneous Thomas Cluster model (Model 6) resulted by fitting the point process intensity to the set of covariates of the ppm with the best BIC score (Model 4). 

```{r results='hide', warning=FALSE}

q_mod_5<- kppm(q_ppp ~ 1, "Thomas", method="clik2")

q_mod_6<- kppm(q_ppp, ~  Slope + CI + TPI + Geo + Meta + CVA,
               "Thomas", method="clik2", data = covar)

# Create KPPM lists
kppp_q_models<-list(model5=q_mod_5,model6=q_mod_6)

# Export KPPM summary
write.table(capture.output(print(kppp_q_models)), file="Output/KPP_Models.txt")
```
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-baqh{text-align:center;vertical-align:top}
.tg .tg-amwm{font-weight:bold;text-align:center;vertical-align:top}
</style>
<table class="tg">
<caption> Tab. 3 - Parameters of the Stationary cluster point process Model 5 and of the Inhomogeneous cluster point process Model 6. Kappa= intensity of the parents.</caption>
<thead>
  <tr>
    <th class="tg-amwm">Models</th>
    <th class="tg-amwm">Intensity</th>
    <th class="tg-amwm">Kappa (Intensity of the parents)</th>
    <th class="tg-amwm">Scale (cluster radius)</th>
    <th class="tg-amwm">Mean cluster size (number of offsprings)</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-baqh">Kppm 5</td>
    <td class="tg-baqh">2.505782e-06</td>
    <td class="tg-baqh">1.606992e-08</td>
    <td class="tg-baqh">4.815851e+02</td>
    <td class="tg-baqh">155.5071 points</td>
  </tr>
  <tr>
    <td class="tg-baqh">Kppm 6</td>
    <td class="tg-baqh">Log intensity: ~Slope + CI + TPI + Geo + Meta + CVA</td>
    <td class="tg-baqh">1.66968e-08</td>
    <td class="tg-baqh">4.72056e+02</td>
    <td class="tg-baqh">Mean cluster size: [pixel image]</td>
  </tr>
</tbody>
</table>


The parameters of the two kppm show that the qubbas spatial clusterization is characterised by a very low number of *parent* points and a mean cluster radius of ≈ 475 m (Tab. 3). The *H2* has been addressed applying the L function to Models 6 (Fig. 7).


```{r results='hide'}

# L Function Neyman-Scott Cluster Process

Linhom_2 <- envelope(q_mod_5, fun=Lest,
                     correction="trans", r=seq(0,1500,50), nsim=NuSim)

Linhom_3 <- envelope(q_mod_6, fun=Lest,
                     correction="trans", r=seq(0,1500,50), nsim=NuSim)

```

```{r echo=FALSE, results='hide'}

# Plot (Fig. 7), L functions (mod. 5 & mod 6)

png(width=2000,height=1000,units="px",res=150, file="Output/Results of the L function on model 5 and model 6.png")
par(mar=c(4,4,4,4))
par(mfrow=c(1,2))
plot(Linhom_2,main="L function (Mod_5)",xlab="Distance (m)", ylab="L function (r)",
     legend=F)
legend("topleft",legend=c("Observed","Expected","Confidence Envelope"),lty=c(1,2,NA),
       col=c(1,2,"lightgrey"),lwd=c(1,1,NA),pch=c(NA,NA,15),cex=0.8,pt.cex=1.5,
       y.intersp=1.5)
plot(Linhom_3,main="L function (Mod_6)",xlab="Distance (m)", ylab="L function(r)",
     legend=F)
legend("topleft",legend=c("Observed","Expected","Confidence Envelope"),lty=c(1,2,NA),
       col=c(1,2,"lightgrey"),lwd=c(1,1,NA),pch=c(NA,NA,15),cex=0.8,pt.cex=1.5,
       y.intersp=1.5)
dev.off()
par(mfrow=c(1,1))
```
The results display no deviations of the observed values from the confidence envelope. This suggests that the distribution of qubbas can be accounted for by broad landscape preferences with local tendencies towards tomb clustering.

# Part 3: Residuals values measure

Finally, the goodness-of-fit of Model 6 was assessed through the calculation of the model's residual values. The diagnostic smoothed residual values of the Model 6 have been calculated through the spatstat function *residuals.kppm*.

```{r results='hide'}
resv <- residuals.kppm(q_mod_6, drop=T)
```

```{r echo=FALSE, results='hide'}

# Plot Fig. 8, Smoothed Residual Values

Valresv<- fortify(data.frame(Smooth(resv)))
Valresv[,3]<-Valresv[,3]*10^7

png(width=1500,height=2000,res = 300, file="Output/Residual values of model 6.png")

ValRes <- ggplot(Valresv, aes(x=x, y=y, z=value)) + geom_tile(aes(fill = value))+
  scale_fill_distiller(type = "seq",palette = "Spectral", 
                       guide = "colourbar", aesthetics = "fill")+
  stat_contour(colour="grey")+
  labs(title = "Residual Values", adj= 1, x = "",y = "")+
  coord_fixed()
print(ValRes)
dev.off()
```

As shown in Fig. 8, the smoothed residual values of Model 6 display an overall uniform level of prediction, except for a limited portion in the centre of the ROI in which the model greatly underestimated the true intensity of sites. The area corresponds to a small outcrop, where the remote survey documented a markedly higher occurrence of qubbas than in the surroundings (∼1/3rd of the dataset). This significant positive autocorrelation (similar values are close to each other) might imply that the location became a pole of attraction of disproportionate size against the others in the region.

# Part 4: Assessing spatial correlation between qubbas and ancient tumuli

Likely, social memory-driven choices may also emerge assessing the possibility of a cultural continuity between tumuli and qubbas. In fact, the CVA coefficient of Model 4 shows that there is a slight direct correlation between the distribution of qubbas and the intervisibility with tumuli (Tab. 2). 

```{r results='hide'}

# Create marked ppp with Tumuli and Qubbas

valT<-paste(rep("T",length(sites_t)))
valQ<-paste(rep("Q",length(sites_q)))
val<-c(valT,valQ)

X<-c(coordinates(sites_t)[,1],coordinates(sites_q)[,1])
Y<-c(coordinates(sites_t)[,2],coordinates(sites_q)[,2])

sites_tq<-ppp(X,Y,region,marks=as.factor(val))
rjitter(sites_tq,0.5)

# Cross G- Fucntion

D <- density(split(sites_tq))

xgf <- envelope(sites_tq, Gcross, i= "Q", j="T", r=seq(0,1500,50), 
                simulate= expression(rmpoispp(D, types=names(D))), nsim = NuSim)

# Extracting the Cross G function highest observed value
print(xgf$r[xgf$obs>xgf$hi])
```

The resulting graph of the cross-type nearest-neighbour function Gij(r) shows that for great distances (>400m) the cross G function displays a segregation pattern that is probably related to the dispersal of the qubbas in the ROI towards locations where the tumuli are not found (Fig. 9). On the contrary, at shorter distances (<400m) there is an aggregation of the qubbas around the tumuli that is consistent throughout the region (Fig. 9).

```{r echo=FALSE, results='hide'}

# Plot (Fig. 9) Cross G function

png(width=2500,height=2000,units="px",res=300, file="Output/The results of the cross G function.png")
plot(xgf,main="Cross G function",xlab="Distance (m)", ylab="G q,t (r)",
     legend=F)
legend("topleft",legend=c("Observed","Expected","Confidence Envelope"),lty=c(1,2,NA),
       col=c(1,2,"lightgrey"),lwd=c(1,1,NA),pch=c(NA,NA,15),cex=0.8,pt.cex=1.5,
       y.intersp=1.5)
dev.off()
```
