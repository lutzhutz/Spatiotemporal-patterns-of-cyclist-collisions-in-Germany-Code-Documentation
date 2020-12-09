---
title: Analysis of Collisions with Cyclist Participation by Regional Typology in Germany
  Code Documentation
output:
  html_document:
    keep_md: true
  word_document: default
editor_options:
  chunk_output_type: console
---
This is the R Skript corresponding to the Bachelor Thesis "Analysis of Collisions with Cyclist Participation by Regional Typology in Germany". In the first section the functions for downloading and processing collision data from the collision dataset of the German Federal Statistical Office will be presented and explained step by step. The sections that follow represent the workflow for all statistics and graphs that are included in the thesis. To run the full script at once **you will need the b_osm.xlsx file** provided in the GitHub repository [Analysis of Collisions with Cyclist Participation by Regional Typology in Germany Code Documentation](https://github.com/lutzhutz/Analysis-of-Collisions-with-Cyclist-Participation-by-Regional-Typology-in-Germany-Code-Documentation) starting from the section *"Nearest road maximum speed limit"* as some of the steps have been made in the software QGIS. 

Note: The library [collisionsDE](https://github.com/lutzhutz/collisionsDE) is a self written library by the author. The Bachelor Thesis is available at https://www.researchgate.net/profile/Lasse_Harkort. 

#Libraries
```{r}
library("devtools")# for downloading GitHub packages
#install_github("lutzhutz/collisionsDE")
library(collisionsDE)
#install.packages("colorspace")
library(colorspace)
#install.packages("dplyr)
library(dplyr)
#install.packages("sf")
library(sf)
#install.packages("ggplot2")
library(ggplot2)
#install.packages("lemon")
library(lemon)
#install.packages("tidyr")
library(tidyr)
#install.packages("stringr")
library(stringr)
#install.packages("ggpubr")
library(ggpubr)
#install.packages("grid")
library(grid)
#install.packages("lwgeom")
library(lwgeom)
#install.packages("readxl")
library(readxl)

```

Legend for data name extensions (chronologically):

- y.## = year (e.g. 2018 -> y.19)
- .df = data frame of all collisions
- .b = all collisions with at least one cyclist involved
- .f = frequency
- .k = killed
- .si = seriously injured
- .sli = slightly injured
- .sev = severity
- .ct = collision type
- .ck = collision - killed
- .c = collisions
- .m = month
- .h = hours (daytime)
- .w = weekdays
- .osm = highway data from Open Street Map
- .s = subset
- .m = maxspeed categories
- .mot = motorized
- .g = grouped
- .wi = with information (roads with information on maximum speed limit)
- .ni = no information (roads with no information on maximum speed limit)
- .mfc = merged feature classes (merged datasets of .wi and .ni)


#Pre-processing
```{r}
#disable e format
options(scipen = 999)

#import all reported collisions with personal injury from 2018 (function from collisionsDE)
#y.19<-import_2018()
y.19<-import_2019()

#subset all accident with at least one cyclist involved
y.19.b<- y.19[y.19$IstRad == 1,]

#add regional information to the collision events (function from collisionsDE)
y.19<-add_regions(y.19)
y.19.b<-add_regions(y.19.b)

#write_sf(y.19.b,"C:/Users/LH/Desktop/Uni/B.Sc/OSM_Highway_19/Unfall_19_proj.shp","Unfall_19_b")

#transform from spatial feature object to data frame
y.19.df.b <- y.19.b %>% st_drop_geometry()

y.19.df <- y.19 %>% st_drop_geometry()
#subset all accident with at least one cyclist involved
#y.19.df.b <- y.19.df[y.19.df$IstRad == 1,]

#add regional information to the collision events (function from collisionsDE)
#y.19.df.b<-add_regions(y.19.df.b)

y.19.df.b<-y.19.df.b[!is.na(y.19.df.b$regio7bez),]

y.19.df<-y.19.df[!is.na(y.19.df$regio7bez),]

#data frame overview
head(y.19.df.b)

#suppress note message from dplyr
options(dplyr.summarise.inform=F) 

#y.19$ID <- seq.int(nrow(y.19))

#write_sf(y.19,"C:/Users/LH/Desktop/Uni/B.Sc/OSM_Highway_19/Unfall_19_proj.shp","Unfall_19_proj")
```

#General overview
```{r}
#data transform
y.19.o <-
  y.19.df %>%
  mutate(countb = if_else(y.19.df$IstRad == 1, 1, 0)) %>%
  group_by(gemname, regio7bez, gembev, gemfl) %>%
  summarise(count = sum(n()), countb = sum(countb)) %>%
  group_by(regio7bez) %>%
  summarise(
    gsm_fl = sum(gemfl),
    gsm_bev = sum(gembev),
    count = sum(count),
    countb = sum(countb),
    popdens = round(gsm_bev / gsm_fl,0),
  ) %>%
  mutate(
    gsm_fl = gsm_fl / sum(gsm_fl) * 100,
    gsm_bev = gsm_bev / sum(gsm_bev) * 100
    #shareb = countb / sum(countb) * 100
  ) %>%
  pivot_longer(c(gsm_fl, gsm_bev, count, countb, popdens, gsm_fl, gsm_bev))

manlabel<-c("","","(19.9)","(25.9)","","","","(15.2)","(18.7)","","","","(24.3)","(24)","","","","(5.9)","(3.3)","","","","(6.6)","(8.1)","","","","(13.8)","(12.5)","","","","(14.3)","(7.6)","")

#plot export in 1920x (maintain ratio)
ggplot(transform(y.19.o, name = factor(
  name,
  levels = c("gsm_fl", "gsm_bev", "popdens", "count", "countb"),
  labels = c(
    "Area in %",
    "Population in %",
    "Population density (pop/km²)",
    "Number of collisions (% share)",
    "With cyclists participation (% share)"
  )
)),
aes(regio7bez, value, fill = regio7bez)) +
  geom_bar(stat = "identity") +
  geom_text(aes(label = round(value, 1)), vjust = -0.4, size = 2.7) +
  geom_text(aes(label = manlabel), vjust = 1.5, size = 2.7)+
  facet_wrap(~ name, scales = "free") +
  theme_light()+
  theme(
    axis.title = element_blank(),
    axis.text.x = element_blank(),
    axis.ticks.x = element_blank(),
    axis.text.y = element_blank(),
    axis.ticks.y = element_blank(),
    legend.position = "bottom",
    legend.title = element_blank(),
    legend.text = element_text(size = 9),
    strip.text = element_text(color = 'black',size = 9, face = "italic"),
    strip.background = element_blank()
  ) +
  scale_fill_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City",
      "Small-town Area",
      "Central City",
      "Medium-sized City",
      "Small-town Area"
    )
  ) +
  guides(fill = guide_legend(nrow = 1))

ggsave("overview.png",dpi = 300,width = 8.83,height = 7.15)
#Saving 8.83 x 7.15 in image
getwd()
```

#Collision frequency
```{r}
# Here the values of table 2 in the thesis are created

y.19.f <-
  y.19.df %>%
  mutate(countb = if_else(y.19.df$IstRad == 1, 1, 0)) %>% #indicate all collisions where at least one cyclist has been involved
  group_by(gemname, regio7bez, gembev, gemfl) %>%
  summarise(count = sum(n()),
            countb = sum(countb)) %>% #sum up the number of all collisions, the number of collisions with cyclist participation, population and area
  group_by(regio7bez) %>% #sum these up again to the seven regional levels
  summarise(
    gsm_fl = sum(gemfl), #total area
    gsm_bev = sum(gembev), #total population
    count = sum(count), #number of collisions
    countb = sum(countb), #number of collisions with cyclist participation
    popdens = gsm_bev / gsm_fl,
    .groups = "drop" #population density
  ) %>%
  mutate(
    gsm_fl_s = gsm_fl / sum(gsm_fl) * 100,
    gsm_bev_s = gsm_bev / sum(gsm_bev) * 100,
    share_s = count / sum(count) * 100,
    shareb_s = countb / sum(countb) * 100,
    acc_pers = count * 10000 / gsm_bev,
    acc_persb = countb * 10000 / gsm_bev #calculate the shares of each of the variables calculated before as well as the collisions per 10.000 residents of each region
  )

round(y.19.f$acc_persb, 1) #round the numbers to the first decimal

cor.test(y.19.f$popdens, y.19.f$acc_pers, method = "pearson") #calculate Pearson correlation
cor.test(y.19.f$popdens, y.19.f$acc_persb, method = "pearson") #calculate Pearson correlation
```

#Collision severity
```{r}
y.19.df.k <-
  y.19.df.b[y.19.df.b$UKATEGORIE == 1, ] #subset all collisions with at least one person killed

#transform data
y.19.df.k <- y.19.df.k %>%
  group_by(regio7bez) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100,
         cat = "killed") #calculate the shares of all fatal crashes by each region

y.19.df.si <-
  y.19.df.b[y.19.df.b$UKATEGORIE == 2, ] #subset all collisions with at least one person seriously injured

#transform data
y.19.df.si <- y.19.df.si %>%
  group_by(regio7bez) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100,
         cat = "seriously_injured") #calculate the shares of all crashes with at least one person seriously injured by each region

y.19.df.sli <-
  y.19.df.b[y.19.df.b$UKATEGORIE == 3, ] #subset all collisions with at least one person slightly injured

#transform data
y.19.df.sli <- y.19.df.sli %>%
  group_by(regio7bez) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100,
         cat = "injury") #calculate the shares of all crashes with at least one person slightly injured by each region


y.19.df.sev <-
  rbind(y.19.df.k, y.19.df.si, y.19.df.sli) #combine the three created datasets

#check total counts
sum(y.19.df.k$count)
sum(y.19.df.si$count)
sum(y.19.df.sli$count)

labels = c(
        "Metropolis",
        "Regiopolis",
        "Medium-sized City\n(u)",
        "Small-town Area\n(u)",
        "Central City",
        "Medium-sized City\n(r)",
        "Small-town Area\n(r)"
      )



#visualization (Figure 9 in Thesis)
ggplot(
  transform(
    y.19.df.sev,
    cat = factor(
      cat,
      levels =
        c("injury",
          "seriously_injured",
          "killed"),
      label = c(
        "Slightly Injured (100% = 61.221)",
        "Seriously injured (100% = 12.928)",
        "Fatal (100% = 382)"
      )
    ),
    regio7bez = factor(
      regio7bez,
      levels =
        c(
          "U_Metro",
          "U_Regiop",
          "U_Medium",
          "U_Small",
          "R_C",
          "R_Med_C",
          "R_Small_C"
        ),
      label = labels
    )
  ),
  aes(regio7bez, share, fill = cat)
) +
  geom_bar(stat = "identity",
           position = "dodge",
           width = 0.8) +
  geom_text(
    aes(label = round(share, 1)),
    position = position_dodge(width = 0.8),
    vjust = -0.2,
    color = "gray30",
    size = 3.5
  ) +
  ylab("Share of reported collisions with cyclist participation in each category in %") +
  theme_light() +
  scale_fill_manual("",
                    values = c("grey80",
                               "grey60",
                               "grey40")) +
  theme(
    axis.title.x = element_blank(),
    axis.text.x = element_text(colour = "black", size = 10.5),
    legend.position = "top",
    legend.justification = "left",
    legend.text = element_text(size = 10.5)
  ) +
  annotate("text",
           label = "* u=urban, r=rural",
           x = 7,
           y = 30)


#save in output workspace
ggsave("severity2.png", dpi = 300, width = 9.1, height = 7.15)
```

#Collision type
```{r}
#calculate share of each collision type by region
y.19.df.b.ct <- y.19.df.b %>%
  group_by(regio7bez, UTYP1) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100) 

#visualize (Figure 10 in thesis)
type<-ggplot(
    transform(
    y.19.df.b.ct,
    UTYP1 = factor(
      UTYP1,
      levels = c(
        "3",
        "2",
        "1",
        "7",
        "6",
        "5",
        "4"
      ),
      label = c(
        "collision when turing into/crossing",
        "collision when turning off",
        "driving collision",
        "other collision",
        "collision while moving along",
        "collision with stationary vehicle",
        "collision while pedestrian crossing"
      )
    ),
    regio7bez = factor(
      regio7bez,
      levels =
        c(
          "R_Small_C",
          "R_Med_C",
          "R_C",
          "U_Small",
          "U_Medium",
          "U_Regiop",
          "U_Metro"
        ),
      label = c(
        "Small-town Area (r)",
        "Medium-sized City (r)",
        "Central City",
        "Small-town Area (u)",
        "Medium-sized City (u)",
        "Regiopolis",
        "Metropolis"
      )
    )
  ),
  aes(x = share, y = regio7bez)
) +
  facet_wrap( ~ UTYP1, ncol = 4)  +
  geom_bar(stat = "identity", fill = rep(c("#B42A5D",
    "#FE2F7C",
    "#FF7776",
    "#FFE0DC",
    "#00858B",
    "#41B8B9",
    "#D2E38C"
  ),7)) +
  geom_text(aes(label = round(share,1)), hjust = -0.08, size = 3) +
  theme_light() +
  scale_x_continuous(limits = c(0, 42)) +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(color = 'black',size = 10),
    strip.text = element_text(color = 'black',size = 10, face = "italic"),
    strip.background = element_blank()
  )

#check total counts
y.19.df.b.ct %>%
  group_by(regio7bez) %>%
  summarise(count = sum(count))

#add counts to ggplot graph
print(type)
grid.text(
  paste0(
    "Metropolis = 19,267",
    "\n",
    "Regiopolis = 13,927",
    "\n",
    "Medium-sized City (u) = 17,853",
    "\n",
    "Small-town Area (u) = 2,475",
    "\n",
    "Central City  = 6027",
    "\n",
    "Medium-sized City (r) = 9,284",
    "\n",
    "Small-town Area (r) = 5,698",
    "\n",
    "\n",
    "*u=urban, r=rural"
  ),
  x = 0.79,
  y = 0.32,
  just = "left",
  gp = gpar(fontsize = 10)
)

#export:Save as image .. 1122x590
```

#Collision category


```{r}
#transform
y.19.df.b.cc <- y.19.df.b %>%
  group_by(regio7bez, UART, ) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)
```


#Collisions between cyclists and other parties with fatal outcome
```{r}
#subset all collisions with at least one person killed
y.19.df.b.ck <- y.19.df.b[y.19.df.b$UKATEGORIE == 1,]

#add collision type with collision_types() (function from collisionsDE)
y.19.df.b.ck <- collision_types(y.19.df.b.ck)

#group by region and collision type, compute shares
y.19.df.b.ck <- y.19.df.b.ck %>%
  group_by(regio7bez, coll_typ) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualize (Figure 11 in the thesis)
b<-ggplot(
  transform(
    y.19.df.b.ck,
    coll_typ = factor(
      coll_typ,
      levels = c(
        "bicyclecar",
        "bicycle",
        "bicyclefoot",
        "bicycleother",
        "bicyclemcycle",
        "bicycletruck",
        "three"
      ),
      labels = c(
        "bicycle-car",
        "bicycle only accidents",
        "bicycle-foot",
        "bicycle-other parties",
        "bicycle-motorcycle",
        "bicycle-truck",
        "more than two participants"
      )
    ),
    regio7bez = factor(
      regio7bez,
      levels =
        c(
          "R_Small_C",
          "R_Med_C",
          "R_C",
          "U_Small",
          "U_Medium",
          "U_Regiop",
          "U_Metro"
        ),
      label = c(
        "Small-town Area (r)",
        "Medium-sized City (r)",
        "Central City",
        "Small-town Area (u)",
        "Medium-sized City (u)",
        "Regiopolis",
        "Metropolis"
      )
    )
  ),
  aes(x = share, y = regio7bez)
) +
  facet_wrap( ~ coll_typ, ncol = 4)  +
  geom_bar(
    stat = "identity",
    fill = c(
      "#B42A5D",#1
      "#FE2F7C",#2
      "#FF7776",#3
      "#FFE0DC",#4
      "#00858B",#5
      "#41B8B9",#6
      "#D2E38C",#7
      "#B42A5D",#1
      "#FE2F7C",#2
      "#FF7776",#3
      "#FFE0DC",#4
      "#00858B",#5
      "#41B8B9",#6
      "#D2E38C",#7
      "#FE2F7C",#2
      "#FF7776",#3
      "#41B8B9",#6
      "#B42A5D",#1
      "#FE2F7C",#2
      "#FF7776",#3
      "#FFE0DC",#4
      "#00858B",#5
      "#41B8B9",#6
      "#D2E38C",#7
      "#FF7776",#3
      "#FFE0DC",#4
      "#00858B",#5
      "#41B8B9",#6
      "#D2E38C",#7
      "#B42A5D",#1
      "#FE2F7C",#2
      "#FF7776",#3
      "#FFE0DC",#4
      "#00858B",#5
      "#41B8B9",#6
      "#D2E38C",#7
      "#FE2F7C",#2
      "#FF7776",#3
      "#FFE0DC",#4
      "#D2E38C"#7
    )
  ) +
  geom_text(aes(label = round(share, 1)), hjust = -0.08, size = 3) +
  theme_light() +
  scale_x_continuous(limits = c(0, 76)) +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(color = 'black', size = 10),
    strip.text = element_text(color = 'black', size = 10, face = "italic"),
    strip.background = element_blank()
  )

#check total counts
y.19.df.b.ck %>%
  group_by(regio7bez) %>%
  summarise(count = sum(count))

#add counts to ggplot graph
print(b)
grid.text(
  paste0(
    "Metropolis = 44",
    "\n",
    "Regiopolis = 46",
    "\n",
    "Medium-sized City (u) = 104",
    "\n",
    "Small-town Area (u) = 25",
    "\n",
    "Central City = 24",
    "\n",
    "Medium-sized City (r) = 68",
    "\n",
    "Small-town Area (r) = 71",
    "\n",
    "\n",
    "*u=urban, r=rural"
  ),
  x = 0.79,
  y = 0.32,
  just = "left",
  gp = gpar(fontsize = 10)
)

#export:Save as image .. 1122x590
```

#Collisions between cyclists and other parties, all collisions
```{r}
#add collision type with collision_types() function by collisionsDE
y.19.df.b.c <- collision_types(y.19.df.b)

#group by region and collision type, compute shares
y.19.df.b.c <- y.19.df.b.c %>%
  group_by(regio7bez, coll_typ) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualize (Figure 12 in the thesis)
a <- ggplot(
  transform(
    y.19.df.b.c,
    coll_typ = factor(
      coll_typ,
      levels = c(
        "bicyclecar",
        "bicycle",
        "bicyclefoot",
        "bicycleother",
        "bicyclemcycle",
        "bicycletruck",
        "three"
      ),
      labels = c(
        "bicycle-car",
        "bicycle only accidents",
        "bicycle-foot",
        "bicycle-other parties",
        "bicycle-motorcycle",
        "bicycle-truck",
        "more than two participants"
      )
    ),
    regio7bez = factor(
      regio7bez,
      levels =
        c(
          "R_Small_C",
          "R_Med_C",
          "R_C",
          "U_Small",
          "U_Medium",
          "U_Regiop",
          "U_Metro"
        ),
      label = c(
        "Small-town Area (r)",
        "Medium-sized City (r)",
        "Central City",
        "Small-town Area (u)",
        "Medium-sized City (u)",
        "Regiopolis",
        "Metropolis"
      )
    )
  ),
  aes(x = share, y = regio7bez)
) +
  facet_wrap(~ coll_typ, ncol = 4)  +
  geom_bar(stat = "identity", fill = rep(
    c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    7
  )) +
  geom_text(aes(label = round(share, 1)), hjust = -0.08, size = 3) +
  theme_light() +
  scale_x_continuous(limits = c(0, 70)) +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(color = 'black', size = 10),
    strip.text = element_text(color = 'black', size = 10, face = "italic"),
    strip.background = element_blank()
  )

#check total counts
y.19.df.b.c %>%
  group_by(regio7bez) %>%
  summarise(count = sum(count))

#add counts to ggplot graph
print(a)
grid.text(
  paste0(
    "Metropolis = 19.267",
    "\n",
    "Regiopolis = 13.927",
    "\n",
    "Medium-sized City (u) = 17.853",
    "\n",
    "Small-town Area (u) = 2.475",
    "\n",
    "Central City = 6.027",
    "\n",
    "Medium-sized City (r) = 9.284",
    "\n",
    "Small-town Area (r)  = 5.698",
    "\n",
    "\n",
    "*u=urban, r=rural"
  ),
  x = 0.79,
  y = 0.32,
  just = "left",
  gp = gpar(fontsize = 10)
)

#export:Save as image .. 1122x590
```

#Temporal distribution
```{r}
#calculate shares of collisions per month by region, compute shares
y.19.df.b.m <- y.19.df.b %>%
  group_by(regio7bez, UMONAT) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for month (warning message can be ignored)
m <-
  ggplot(y.19.df.b.m,
         aes(UMONAT, share, group = regio7bez, color = regio7bez)) +
  geom_line(size = 0.7) +
  theme_bw() +
  ggtitle("Month") +
  scale_x_discrete(
    labels = c(
      "Jan",
      "Feb",
      "Mar",
      "Apr",
      "May",
      "Jun",
      "Jul",
      "Aug",
      "Sep",
      "Oct",
      "Nov",
      "Dez"
    )
  ) +
  scale_colour_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City (u)",
      "Small-town Area (u)",
      "Central City",
      "Medium-sized City (r)",
      "Small-town Area (r)"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#calculate shares of collisions per weekday by region, compute shares
y.19.df.b.w <- y.19.df.b %>%
  group_by(regio7bez, UWOCHENTAG) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for weekdays
w <- ggplot(y.19.df.b.w,
            aes(UWOCHENTAG, share, group = regio7bez, color = regio7bez)) +
  geom_line(size = 0.7) +
  theme_bw() +
  ggtitle("Weekdays") +
  scale_x_discrete(
    limits = c(2, 3, 4, 5, 6, 7, 1),
    labels = c("Tue",
               "Wed",
               "Thu",
               "Fri",
               "Sat",
               "Sun",
               "Mon")
  ) +
  scale_colour_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City",
      "Small-town Area",
      "Central City",
      "Medium-sized City",
      "Small-town Area"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#extract only working days for the hourly shares of collision frequency
y.19.df.b.working <-
  y.19.df.b[!(y.19.df.b$UWOCHENTAG == 7 | y.19.df.b$UWOCHENTAG == 1),]

#calculate shares of collisions per daytime by region, compute shares
y.19.df.b.h <- y.19.df.b.working %>%
  mutate(
    #revalue to daytime categories
    hourcat = case_when(
      USTUNDE %in% c('02', '03', '04', '05') ~ 'night_m',
      USTUNDE %in% c('06', '07', '08', '09') ~ 'morning',
      USTUNDE %in% c('10', '11', '12', '13') ~ 'noon',
      USTUNDE %in% c('14', '15', '16', '17') ~ 'afternoon',
      USTUNDE %in% c('18', '19', '20', '21') ~ 'evening',
      USTUNDE %in% c('22', '23', '00', '01') ~ 'night'
    )
  ) %>%
  group_by(regio7bez, hourcat) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for daytimes
h <- ggplot(y.19.df.b.h,
            aes(hourcat, share, group = regio7bez, color = regio7bez)) +
  geom_line(size = 0.7) +
  theme_bw() +
  ggtitle("Daytime*") +
  scale_x_discrete(
    limits = c('night_m',
               'morning',
               'noon',
               'afternoon',
               'evening',
               'night'),
    labels = c(
      "early monring (2-6am)",
      "morning (6-10am)",
      "noon (10am - 1pm)",
      "afternoon (2-6pm)",
      "evening (6-10pm)",
      "night (10pm-2am)"
    )
  ) +
  scale_colour_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City",
      "Small-town Area",
      "Central City",
      "Medium-sized City",
      "Small-town Area"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#arrange all plots to one graph (Figure 13 in the thesis)
figure <-
  ggarrange(
    m,
    w,
    h,
    ncol = 1,
    nrow = 3,
    common.legend = T,
    legend = "bottom"
  )
annotate_figure(
  figure,
  left = text_grob(
    "Share of all collisions with cyclists participation per region in %",
    rot = 90,
    size = 9
  ),
  bottom = text_grob(
    paste0("u=urban, r=rural      *weekends excluded"),
    hjust = 1.2,
    x = 1,
    face = "italic",
    size = 9
  )
)

#save graph as image
ggsave(
  "C:/Users/LH/Desktop/Uni/B.Sc/R/month_hour_week22.png",
  width = 8.04,
  height = 6.99,
  dpi = 300
) 

```

#Temporal distribution bars
```{r}
#calculate shares of collisions per month by region, compute shares
y.19.df.b.m <- y.19.df.b %>%
  group_by(regio7bez, UMONAT) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for month (warning message can be ignored)
m<-ggplot(y.19.df.b.m,aes(UMONAT, share,fill=regio7bez)) +
  geom_bar(stat = "identity",width = 0.7, position = "dodge") +
  theme_bw() +
  ggtitle("Month") +
  scale_x_discrete(
    labels = c(
      "Jan",
      "Feb",
      "Mar",
      "Apr",
      "May",
      "Jun",
      "Jul",
      "Aug",
      "Sep",
      "Oct",
      "Nov",
      "Dez"
    )
  ) +
  scale_fill_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City (u)",
      "Small-town Area (u)",
      "Central City",
      "Medium-sized City (r)",
      "Small-town Area (r)"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#calculate shares of collisions per weekday by region, compute shares
y.19.df.b.w <- y.19.df.b %>%
  group_by(regio7bez, UWOCHENTAG) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for weekdays
w<-ggplot(y.19.df.b.w,
            aes(UWOCHENTAG, share, fill = regio7bez)) +
  geom_bar(stat = "identity",width = 0.7, position= "dodge") +
  theme_bw() +
  ggtitle("Weekdays") +
  scale_x_discrete(
    limits = c(2, 3, 4, 5, 6, 7, 1),
    labels = c("Tue",
               "Wed",
               "Thu",
               "Fri",
               "Sat",
               "Sun",
               "Mon")
  ) +
  scale_fill_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City (u)",
      "Small-town Area (u)",
      "Central City",
      "Medium-sized City (r)",
      "Small-town Area (r)"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#extract only working days for the hourly shares of collision frequency
y.19.df.b.working <-
  y.19.df.b[!(y.19.df.b$UWOCHENTAG == 7 | y.19.df.b$UWOCHENTAG == 1),]

#calculate shares of collisions per daytime by region, compute shares
y.19.df.b.h <- y.19.df.b.working %>%
  mutate(
    #revalue to daytime categories
    hourcat = case_when(
      USTUNDE %in% c('02', '03', '04', '05') ~ 'night_m',
      USTUNDE %in% c('06', '07', '08', '09') ~ 'morning',
      USTUNDE %in% c('10', '11', '12', '13') ~ 'noon',
      USTUNDE %in% c('14', '15', '16', '17') ~ 'afternoon',
      USTUNDE %in% c('18', '19', '20', '21') ~ 'evening',
      USTUNDE %in% c('22', '23', '00', '01') ~ 'night'
    )
  ) %>%
  group_by(regio7bez, hourcat) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualization for daytimes
h<-ggplot(y.19.df.b.h,
            aes(hourcat, share, fill= regio7bez)) +
  geom_bar(stat = "identity", width = 0.7, position= "dodge") +
  theme_bw() +
  ggtitle("Daytime*") +
  scale_x_discrete(
    limits = c('night_m',
               'morning',
               'noon',
               'afternoon',
               'evening',
               'night'),
    labels = c(
      "Early morning (2-6am)",
      "Morning (6-10am)",
      "Noon (10am - 1pm)",
      "Afternoon (2-6pm)",
      "Evening (6-10pm)",
      "Night (10pm-2am)"
    )
  ) +
  scale_fill_manual(
    "legend",
    values = c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    labels = c(
      "Metropolis",
      "Regiopolis",
      "Medium-sized City (u)",
      "Small-town Area (u)",
      "Central City",
      "Medium-sized City (r)",
      "Small-town Area (r)"
    )
  ) +
  theme(
    plot.title = element_text(hjust = 0.5, size = 10, face = "bold"),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 9),
    axis.text.y = element_text(size = 9),
    legend.position = "bottom",
    legend.title = element_blank()
  )

#arrange all plots to one graph (Figure 13 in the thesis)
figure <-
  ggarrange(
    m,
    w,
    h,
    ncol = 1,
    nrow = 3,
    common.legend = T,
    legend = "bottom"
  )
annotate_figure(
  figure,
  left = text_grob(
    "Share of all collisions with cyclists participation per region in %",
    rot = 90,
    size = 9
  ),
  bottom = text_grob(
    paste0("u=urban, r=rural      *weekends excluded"),
    hjust = 1.2,
    x = 1,
    face = "italic",
    size = 9
  )
)

#save graph as image
ggsave(
  "month_hour_week22.png",
  width = 8.04,
  height = 6.99,
  dpi = 300
) 

```





WARNING: this step takes approximately 45min. It is recommended to use the processed file provided in the folder

However, if you would like to reproduce this step you need to specify the outputs of the shapefiles that will be downloaded (note: combined size of all shapefiles is about 8GB)

#Download osm highway dataset
```{r}
#set output directory - needs to be specified!
outdir="C:/Users/LH/Desktop/Uni/B.Sc/OSM_Highway_19"

#function for downloading osm highway data from Germany
dwl_state_highway <- function(state,output) {
  temp <- tempfile()
  download.file(
    paste0(
      "http://download.geofabrik.de/europe/germany/",
      state,
      "-latest-free.shp.zip"
    ),
    temp
  )
  unzip(temp)
  state_data <-
    st_read("./gis_osm_roads_free_1.shp", "gis_osm_roads_free_1")
  state_data$length<-st_length(state_data)
  output=output
  write_sf(state_data,output,state,driver = "ESRI Shapefile")
}

#download osm highway data by federal state - the data for all of Germany would have been to huge in size
dwl_state_highway(state = "bremen", output = outdir)
dwl_state_highway(state = "hamburg", output = outdir)
dwl_state_highway(state = "schleswig-holstein", output = outdir)
dwl_state_highway(state = "niedersachsen", output = outdir)
dwl_state_highway(state = "bayern", output = outdir)
dwl_state_highway(state = "baden-wuerttemberg", output = outdir)
dwl_state_highway(state = "saarland", output = outdir)
dwl_state_highway(state = "rheinland-pfalz", output = outdir)
dwl_state_highway(state = "hessen", output = outdir)
dwl_state_highway(state = "sachsen", output = outdir)
dwl_state_highway(state = "sachsen-anhalt", output = outdir)
dwl_state_highway(state = "brandenburg", output = outdir)
dwl_state_highway(state = "berlin", output = outdir)
dwl_state_highway(state = "nordrhein-westfalen", output = outdir)
dwl_state_highway(state = "thueringen", output = outdir)
```
To add features from the nearest road of the crash location to each collision event, the algorithm “join attribute by nearest” provided by the geographic information system software QGIS (Version 3.12.2) has been used.

The steps undertaken in QGIS were as follows:

1. merge all federal states to one layer ("merge vector layers")
2. select ETRS89 (Germany) as Destination CRS
2. delete duplicates by scanning osm_id ("delete duplicates by attribute")
4. Download bicycle collision data from https://unfallatlas.statistikportal.de/_opendata2020.html and load it into QGIS
3. project bicycle crash data and osm highway germany to ETRS89 (Germany)
4. create Id field by using rownumber (use "@row_number" as expression)
5. use algorithm "join attributes by nearest" for bicycle crash data to osm highway Germany 
6. export as .csv, the .csv was then imported to excel and saved as .xslx in order to minimize the data size

#Nearest road maximum speed limit **works only with b_osm.xslx file!**
```{r}
#here, the b_osm.xslx file provided in the GitHub repository needs to be put into your workspace (if you prefer to have it elsewhere the read_excel directory needs to be updated as well)

# y.19.df.b.osm <-
#   read.csv("C:/Users/LH/Desktop/Uni/B.Sc/OSM_Highway_19/b_osm_19.csv")


#all 252 points that have been found to be the same distance to two line segments are dropped
y.19.df.b.osm.s <-
  y.19.df.b.osm[!(duplicated(y.19.df.b.osm$ID) |
            duplicated(y.19.df.b.osm$ID, fromLast = TRUE)), , drop = FALSE]

#subset all accident with at least one cyclist involved
y.19.df.b.osm.s <- y.19.df.b.osm.s[y.19.df.b.osm.s$IstRad == 1,]

#all points with a distance bigger than 10m to the next road feature are dropped too
y.19.df.b.osm.s <- y.19.df.b.osm.s[y.19.df.b.osm.s$distance < 10, ]

#drop maxspeed that is 0 and empty regio7bez
y.19.df.b.osm.s<-y.19.df.b.osm.s[!y.19.df.b.osm.s$regi7bz =="",]


#mean and median distance from a collision event to the nearest road
round(mean(y.19.df.b.osm.s$distance),2)
round(median(y.19.df.b.osm.s$distance),2)

#classify maximum speed limit into new categories
y.19.df.b.osm.m <- y.19.df.b.osm.s %>%
  mutate(
    maxspeed_c = case_when(
      maxspeed == 0 ~ "n_a",
      maxspeed > 0 & maxspeed <= 30 ~ "_30",
      maxspeed > 30 &
      maxspeed <= 60  ~ "40_60",
      maxspeed > 60 &
      maxspeed <= 90 ~ "70_90",
      maxspeed > 90 ~ "100_"
    )
  )

#subset all collision between cyclist and motorized vehicle
y.19.df.b.osm.mot <- y.19.df.b.osm.m %>%
  #compute TRUE or FALSE for motorized or not motorized vehicle
  mutate(bicyclem = y.19.df.b.osm.m$IstPKW == 1 |
           y.19.df.b.osm.m$IstKrad == 1 |
           y.19.df.b.osm.m$IstGkfz == 1 |
           y.19.df.b.osm.m$IstSnst == 1)


#subset all TRUE
y.19.df.b.osm.mot <- y.19.df.b.osm.mot[y.19.df.b.osm.mot$bicyclem == TRUE, ]

#group all collisions by their nearest road maximum speed limit and the region and calculate the shares
y.19.df.b.osm.mot.g <- y.19.df.b.osm.mot %>%
  group_by(regi7bz, maxspeed_c) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#visualize (Figure 15 in the thesis)
s <- ggplot(
  transform(
    y.19.df.b.osm.mot.g,
    maxspeed_c = factor(
      maxspeed_c,
      levels = c("_30",
                 "40_60",
                 "70_90",
                 "100_",
                 "n_a"),
      labels = c(
        "30 km/h or less",
        "40, 50 or 60 km/h",
        "70, 80 or 90 km/h",
        "100 km/h or more",
        "no information on speed limit"
      )
    ),
    regi7bz = factor(
      regi7bz,
      levels =
        c(
          "R_Small_C",
          "R_Med_C",
          "R_C",
          "U_Small",
          "U_Medium",
          "U_Regiop",
          "U_Metro"
        ),
      label = c(
        "Small-town Area (r)",
        "Medium-sized City (r)",
        "Central City",
        "Small-town Area (u)",
        "Medium-sized City (u)",
        "Regiopolis",
        "Metropolis"
      )
    )
  ),
  aes(x = share, y = regi7bz)
) +
  facet_wrap(~ maxspeed_c, ncol = 3)  +
  geom_bar(stat = "identity", fill = rep(
    c(
      "#00858B",
      "#41B8B9",
      "#D2E38C",
      "#FF7776",
      "#B42A5D",
      "#FE2F7C",
      "#FFE0DC"
    ),
    5
  )) +
  geom_text(aes(label = round(share, 1)), hjust = -0.08, size = 3) +
  theme_light() +
  scale_x_continuous(limits = c(0, 58)) +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(color = 'black', size = 10),
    strip.text = element_text(color = 'black', size = 10, face = "italic"),
    strip.background = element_blank()
  )


#check total counts
y.19.df.b.osm.mot.g %>%
  group_by(regi7bz) %>%
  summarise(count = sum(count))

sum(y.19.df.b.osm.mot.g$count)
#add counts to ggplot graph (Figure 15 in the thesis)
print(s)
grid.text(
  paste0(
    "Metropolis = 3.147",
    "\n",
    "Regiopolis = 4.554",
    "\n",
    "Medium-sized City (u) = 2.130",
    "\n",
    "Small-town Area (u)  = 9.981",
    "\n",
    "Central City  = 9.538",
    "\n",
    "Medium-sized City (r)  = 7.990",
    "\n",
    "Small-town Area (r)  = 991",
    "\n",
    "\n",
    "*u=urban, r=rural"
  ),
  x = 0.72,
  y = 0.32,
  just = "left",
  gp = gpar(fontsize = 10)
)

#export:Save as image .. 1122x590
```

#Nearest road maximum speed limit extrapolation of no information roads
```{r}
#calculations of values from Table 4 in the thesis

#extract all collisions where the nearest road did have a maximum speed limit
y.19.df.b.osm.mot.wi <- y.19.df.b.osm.mot[!y.19.df.b.osm.mot$maxspeed_c == "n_a", ]

#group all road features and calculate the shares
y.19.df.b.osm.mot.wi <- y.19.df.b.osm.mot.wi %>%
  group_by(fclass) %>%
  summarise(count = n()) %>%
  mutate(share = count / sum(count) * 100)

#extract all collisions where the nearest road did not have a maximum speed limit
y.19.df.b.osm.mot.ni <- y.19.df.b.osm.mot[y.19.df.b.osm.mot$maxspeed_c == "n_a", ]

#group all road features and calculate the shares
y.19.df.b.osm.mot.ni <- y.19.df.b.osm.mot.ni %>%
  group_by(fclass) %>%
  summarise(count_na = n()) %>%
  mutate(share_na = count_na / sum(count_na) * 100)

#merge both datasets by feature class
mfc <- merge(y.19.df.b.osm.mot.wi, y.19.df.b.osm.mot.ni, by = "fclass")

#calculate the differences in the shares
mfc$diff <- mfc$share_na - mfc$share

#round up to two decimals
mfc$share_rounded <- round(mfc$share, 2)
mfc$share_na_rounded <- round(mfc$share_na, 2)
mfc$diff_rounded <- mfc$share_na_rounded - mfc$share_rounded

#extract all feature classes that differ either more than 1 or less than -1
mfc.s <- mfc[(mfc$diff >= 1 | mfc$diff <= -1), ]
```

#Trip purpose
```{r}
#create dataset
regio7bez <-
  c("Metropolis",
    "Regiopolis",
    "Medium-sized City (u)",
    "Small-town Area (u)",
    "Central City",
    "Medium-sized City (r)",
    "Small-town Area (r)"
  )

Work <-
  c(26, 25, 22, 18, 23, 23, 17) #data from Mobilität in Tabellen https://mobilitaet-in-tabellen.dlr.de/mit/ , Zeile: Hauptzweck des Weges, Spalte: Zusammengefasster regionalsttistischer Raumtyp, Untergliederung: Verkehrsmittel (Mehrfachantworten-Set) [Accessed: 05.07.2020])
Leisure <-
  c(30, 32, 32, 37, 30, 32, 41) #data from Mobilität in Tabellen https://mobilitaet-in-tabellen.dlr.de/mit/, Zeile: Hauptzweck des Weges, Spalte: Zusammengefasster regionalsttistischer Raumtyp, Untergliederung: Verkehrsmittel (Mehrfachantworten-Set) [Accessed: 05.07.2020])

#create dataframe
df.commuting <- data.frame(regio7bez, Work, Leisure)
df.commuting <- df.commuting %>%
  pivot_longer(c(Work, Leisure))

#visualize (Figure 14 in the thesis)
purpose<-ggplot(transform(df.commuting,
                 regio7bez = factor(
                   regio7bez,
                   levels =
                     c(
                       "Small-town Area (r)",
                       "Medium-sized City (r)",
                       "Central City",
                       "Small-town Area (u)",
                       "Medium-sized City (u)",
                       "Regiopolis",
                       "Metropolis"
                     )
                 )),
       aes(regio7bez, value)) +
  theme_light() +
  geom_bar(stat = "identity",width = 0.9, fill = rep(
    c(
      "#B42A5D",
      "#FE2F7C",
      "#FF7776",
      "#FFE0DC",
      "#00858B",
      "#41B8B9",
      "#D2E38C"
    ),
    2
  )) +
  coord_flip() +
  facet_wrap( ~ name, ncol = 2)  +
  geom_text(aes(label = round(value, 1)), hjust = -0.08, size = 2.7) +
  theme(
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    axis.text.x = element_text(size = 10),
    axis.text.y = element_text(color = 'black', size = 10),
    strip.text = element_text(color = 'black', size = 10, face = "italic"),
    strip.background = element_blank()
  )

#add footnote to ggplot graph
print(purpose)
annotate_figure(purpose,
  bottom = text_grob(
    paste0("u=urban, r=rural      *weekends excluded"),
  x = 0.87,
  y = 3.22,
  just = "left",
  size = 10)
)


# ggsave(
#   "leisure_work.png",
#   dpi = 300
# ) 
#export:Save as image .. 860x404
```
