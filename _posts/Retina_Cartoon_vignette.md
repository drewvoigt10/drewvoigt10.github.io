---
title: "RetinaCartoon"
date: 2021-09-07
permalink: /posts/2021/09/Retina_Cartoon/
tags:
  - retina
  - scRNA-seq
  - RStats
---

## Using R to Create Cartoon Heatmaps

In my research, I have found it really valuable to draw summary
‘cartoon’ figures that summarize some part of anatomy/physiology that I
am studying. While these overview figures have been great for papers and
presentations, I have often wanted to interact with these figures within
R itself. I thought it would be useful to recolor parts of the cartoon
(specifically different cell populations in a tissue) according to some
quantitative characteristic (in my case, this was usually gene expression
information).

My desire to interact with cartoons in R has been especially strong when
analyzing data from single-cell RNA sequencing experiments. When
analyzing this data, it is often useful to visualize gene expression
mapped over a cloud of tSNE or UMAP coordinates. However, my colleagues
and collaborators wouldn’t always find these plots helpful, as the
position of the clusters weren’t always intuitive for someone looking at
the data for the first time. Therefore, I decided to find a way to plot
a cartoon of our tissue of interest (in this example, the choroid, a
vascular connective tissue underlying the retina) and map expression in
each cluster to a cell type depicted in the cartoon. In this vignette, I
will show you how to replicate this process. This will be especially of
interest to folks working with scRNA-seq data, however the first parts
of this vignette can be applied anytime you’d like to interact with an
image in R.

DISCLAIMER: When I embarked on this project, I tried to tackle it in a
single day. The approach works, but requires multiple steps. Sometimes,
I employed simple solutions like for-loops instead of more elegant
mapping commands. I believe this increases readability and shouldn’t
result in decrease in performance (unless you have drawn a
Sistine-chapel-level-of-detail cartoon!!) There are likely cleaner or
simpler approaches to this problem, but I still believe this vignette
has value worth sharing. The following approach may not be the best way,
but it is <u>***a way***</u> to interact with hand-drawn cartoons in R!

## Part 1: Drawing your cartoon

First, we need to create a cartoon drawing. I create my cartoons in
Adobe Illustrator, but one could also create a cartoon in Microsoft
Powerpoint. A couple of notes about these cartoons:

-   Every subcomponent of the cartoon that you want to interact with in
    R needs its own color. For my research, a ‘subcomponent’ means a
    cell type. Therefore, I colored all of the capillary endothelial
    cells in my cartoon the exact same shade of purple.

-   For each object, I added a narrow black outline (this is purely
    stylistic).

-   I saved the cartoon as a .pdf file

I will share the PDF of file of the cartoon used to make this vignette
on my github:
[www.github.com/drewvoigt10/RetinaCartoon/vignette/images/choroid_cartoon.pdf](www.github.com/drewvoigt10/RetinaCartoon/vignette/images/choroid_cartoon.pdf)

In order to read this image into R, we need to convert it to an XML
file. Converting a PDF to an XML file requires generating an
intermediate image in the form of a PostScript File.

## Part 2: Converting the PDF to a PostScript File to an XML file

I used a free online conversion tool to convert the PDF file to a
PostScript File: <https://convertio.co/pdf-ps/>. This is a great tool
(and does not require any sort of sign up), but the free version only
allows for 10 file conversions a day.

Next, I used the grImport package in R to convert the PostScript file to
an XML file. I have a macbook, and grImport requires installation of
‘ghostscript,’ which is not available for MacOS. Therefore, I performed
this step on my University’s High Performance Computing Cluster,
although a free AWS instance would also work well. I’ll also load the
rest of the libraries we will need for this vignette.

``` r
library(grImport)
library(tidyverse)
library(cowplot)
library(grDevices)
library(ggpubr)
library(here)

PostScriptTrace(here("_posts/RetinaCartoon/images/choroid_cartoon.ps"))
```

Running `PostScriptTrace` created a file called choroid_cartoon.ps.xml
in the same directory as my original PostScript file. I transferred this
new XML file back to my local machine (where I also installed the
grImport package). This XML file is also available on my github:
[www.github.com/drewvoigt10/RetinaCartoon/vignette/images/choroid_cartoon.ps.xml](www.github.com/drewvoigt10/RetinaCartoon/vignette/images/choroid_cartoon.ps.xml)

## Part 3: Classifying each cartoon subcomponent in R

Using the `readPicture()` function from the grImport package, we can
read in the XML file into an object class called “PictureFill.” This
object class has slots for x/y coordinates of lines (used to draw an
individual subcomponents in the cartoon) as well as color information,
line width, etc.

``` r
my_cartoon <- readPicture(here("_posts/RetinaCartoon/images/choroid_cartoon.ps.xml"))
```

Let’s look at the structure of an element in this PictureFill object

``` r
str(my_cartoon@paths[1]$path)
```

    ## Formal class 'PictureFill' [package "grImport"] with 9 slots
    ##   ..@ rule     : chr "nonzero"
    ##   ..@ x        : Named num [1:235] 1004 1004 1004 1005 1006 ...
    ##   .. ..- attr(*, "names")= chr [1:235] "move" "line" "line" "line" ...
    ##   ..@ y        : Named num [1:235] 665 664 661 655 646 ...
    ##   .. ..- attr(*, "names")= chr [1:235] "move" "line" "line" "line" ...
    ##   ..@ rgb      : chr "#80CC28"
    ##   ..@ lty      : num(0) 
    ##   ..@ lwd      : num 10
    ##   ..@ lineend  : num 1
    ##   ..@ linejoin : num 1
    ##   ..@ linemitre: num 10

We can see that each path in this PictureFill object has many slots. The
slot that is of most interest to us is the `@rgb` slot, which contains
the HEX color code of the cartoon subcomponent (ie cells).

We want to capture and store all of the colors for all subcomponents
(ie, cells) in my_cartoon. We will use these colors to identify the
index of all cartoon subcomponents (ie, cells). So, we will go through
in a for-loop and pull out the rgb HEX color code for each element in
my_cartoon, storing this in the vector `all_colors`. Note that in the
conversion of our file, HEX color codes can slightly change, so we
unfortunately can’t look up the HEX color codes we used in PPT or Adobe
Illustrator.

``` r
# we will use a quick-and-dirty for loop to grab all colors from our cartoon object 
# and store them in the vector all_colors
all_colors <- c() 

for(i in 1:length(my_cartoon@paths)){
  if(.hasSlot(my_cartoon@paths[i]$path, "rgb")){ ## some parts of the cartoon (like simple 
                                                 ## lines) may not have a color slot, so we add a catch
    
    color <- my_cartoon@paths[i]$path@rgb        ## pluck off the color
    
    all_colors <- c(all_colors, color)           ## add the color to the vector all_colors
    
  } else {                                       ## if the path does not have an associated color, we 
                                                 ## add an empty string
                                                 ## to not throw off the index
    color <- ""
    all_colors <- c(all_colors, color)
  }

}

unique_colors <- data.frame(color = unique(all_colors)) %>% filter(color != "")

# plot the colors so that you can look them up and manually map back to the picture
unique_colors %>% 
  mutate(x_index = 1) %>%
  ggplot(aes(x = x_index, y = color)) +
    geom_tile(aes(fill = color)) +
    scale_fill_manual(values = sort(unique_colors$color))
```

![](/Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-4-1.png)

### Creating our dictionary

Now, we can create a dictionary that maps each color to a cell type. We
have to do this by hand, sadly, by comparing the above ggplot to the
cartoon drawing we generated in Part 1.

``` r
color_celltype_tibble <- tribble(~hex_color, ~celltype,
                                 "#F9B0B0", "smc", 
                                 "#F9A72B", "schwann-mye",  
                                 "#EA8AED", "macrophage-res", 
                                 "#E92822", "artery", 
                                 "#D3AADA", "rpechor-pericyte", 
                                 "#D1B619", "dendritic", 
                                 "#BBE49B", "melanocyte", 
                                 "#B4CBFF", "t-cell",
                                 "#9C1661", "b-cell", 
                                 "#8346EB", "macrophage-inflam", 
                                 "#80CC28", "fibroblast", 
                                 "#782F9A", "choriocapillaris",
                                 "#22275D", "vein"
                                 )
```

### How to re-color individual parts of your cartoon (a brief example)

These next few sections are provided to show you how we can use this
newly-defined dictionary to recolor a specific cell type of interest.
This logic will be folded into the function `generate_cartoon_data()` in
Part 4. In brief, we have a PictureFill object that contains one element
for each subcomponent (individual cell) in our large cartoon. We can
pluck off the HEX color codes from each of these elements and store
these colors in a vector. Then, we can use our newly defined
`color_celtype_tibble` to link each of these colors to a cell type. By
knowing the color of each element in our PictureFill object, we also
know what cell type that element is referencing.

First, we create a function to search through our object for each of the
above colors. When we find a match, we store the index - this index
tells us which cell type is being drawn! We will use this function below
to match colors with indexes in the PictureFill object, using our
newly-defined dictionary to link the cartoon and cell type names
together.

``` r
# returns the indexes of all paths matching each HEX color in a vector
match_colors_and_indexes <- function(color_celltype_tibble, celltype, all_colors){
  # color_celltype_tibble = dictionary of hexcode colors and celltypes
  # celltype = column name in the tibble with the annotated celltype
  # all_colors = all extracted colors from the PictureFill (Cartoon) Object
  celltype_index <- which(color_celltype_tibble[["celltype"]] == celltype)
  my_vector <- which(all_colors == color_celltype_tibble[["hex_color"]][celltype_index])
  return(my_vector)
}
```

Remember that our cartoon has been processed as a PictureFill object,
which is similar to a list. Each element of the list corresponds to a
different contiguous sub-shape in the cartoon. Our goal is to identify
which shape corresponds to which cell type. We can do this by scrolling
through each element of our PictureFill object, finding that object’s
color, and matching that against our color-celltype dictionary using the
above function.

``` r
# We will create an empty list
# Each element in the list corresponds to a different celltype
# Each element in the list is a VECTOR, which corresponds to the path index(es) corresponding 
# to each celltype
celltype_indexes <- vector(mode = "list", length = nrow(color_celltype_tibble))
names(celltype_indexes) <- color_celltype_tibble[["celltype"]]

## This for loop will fill in the indexes for each celltype
for(i in 1:length(celltype_indexes)){
  index_vector <- match_colors_and_indexes(color_celltype_tibble = color_celltype_tibble,
                                           celltype = color_celltype_tibble[["celltype"]][i],
                                           all_colors = all_colors)
  celltype_indexes[[i]] <- index_vector
}

celltype_indexes
```

    ## $smc
    ## [1] 51 53
    ## 
    ## $`schwann-mye`
    ## [1] 11 15 47
    ## 
    ## $`macrophage-res`
    ## [1] 209 212 215 273
    ## 
    ## $artery
    ## [1] 23 25 27 29
    ## 
    ## $`rpechor-pericyte`
    ## [1]  5  7 33
    ## 
    ## $dendritic
    ## [1] 245 248 251 254
    ## 
    ## $melanocyte
    ## [1] 55 57 59
    ## 
    ## $`t-cell`
    ## [1]   3   9  31  45 206
    ## 
    ## $`b-cell`
    ## [1]  80  84  88 202
    ## 
    ## $`macrophage-inflam`
    ## [1] 259 262 265 268
    ## 
    ## $fibroblast
    ## [1]   1 198 200
    ## 
    ## $choriocapillaris
    ## [1] 19 21 35 41 43
    ## 
    ## $vein
    ## [1] 37 39

``` r
celltype_indexes$melanocyte
```

    ## [1] 55 57 59

Hooray! We have finished the hard part. We see from this list of indexes
that the 55th, 57th, and 59th elements in my_cartoon are all
melanocytes. For demonstration, can plot one of these elements with
cowplot and the grImport function `pictureGrob`:

``` r
cowplot::plot_grid(
  pictureGrob(my_cartoon[[55]]) # we can see from celltype_indexes$melanocyte 
                                # that the 55th element of my_cartoon is a melanocyte!
)
```

![](Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-8-1.png)

We can now change the color of this object easily, which will help us
create a heatmap of gene expression

``` r
my_recolored_cartoon <- my_cartoon[[55]]
my_recolored_cartoon@paths$path@rgb <- "pink"

cowplot::plot_grid(
  pictureGrob(my_recolored_cartoon)
)
```

![](Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-9-1.png)

## Part 4: Mapping single-cell expression data to this cartoon

Let’s review what we’ve done so far. We have drawn a cartoon, converted
this cartoon to a xml file (through an intermediate post script file),
extracted all of the colors from this cartoon, created a dictionary that
matches colors with their cell types, and scrolled through the cartoon
to map each element in the PictureFill list to a cell type.

Now the fun begins - we can start recoloring these cells based on gene
expression!

I use the Seurat R package to analyze my scRNA-seq data. Instead of
uploading the entire gigantic Seurat object to this vignette, I will
simply show you how I summarized the expression for each of my cell
types and then upload a subset of this expression matrix to my github
here:
[www.github.com/drewvoigt10/RetinaCartoon/vignette/avg_exp_choroid_vignette.csv](www.github.com/drewvoigt10/RetinaCartoon/vignette/avg_exp_choroid_vignette.csv)

``` r
#Idents(seurat_obj) <- "celltype" # note: names of cell types should EXACTLY match
                                  # the names in color_celltype_tibble above
# avg_exp <- AverageExpression(seurat_obj, assays = "RNA")$RNA %>% 
#  data.frame() %>% 
#  rownames_to_column("gene_name")

# ensure column names (celltypes) EXACTLY match the above dictionary

# saved as avg_exp_choroid_vignette.csv

avg_exp <- read_csv(here("_posts/RetinaCartoon/avg_exp_choroid_vignette.csv"))
avg_exp
```

    ## # A tibble: 14 x 16
    ##    gene_name   artery     vein choriocapillaris unknown_ec `schwann-mye`
    ##    <chr>        <dbl>    <dbl>            <dbl>      <dbl>         <dbl>
    ##  1 SEMA3G     4.42     0.0566          0.286       0.233         0.123  
    ##  2 ACKR1      0.306   20.5             7.06        2.06          0.300  
    ##  3 CA4        0.410    0.957          18.5         0.518         0.0438 
    ##  4 VWF       13.5     40.9            34.1        12.4           0.506  
    ##  5 PLP1       0.0156   0.0178          0.00382     0.0762       16.4    
    ##  6 MLANA      0.0363   0.0253          0.0307      0.689         0.0916 
    ##  7 APOD       0.321    0.475           0.404       4.93          1.30   
    ##  8 ACTG2      0.00127  0.00285         0.000394    0.0142        0      
    ##  9 CCL19      0.0531   0.487           0.186       0.261         0.0527 
    ## 10 CD79A      0.00923  0.00972         0.0226      0.00256       0.00958
    ## 11 CD2        0.00458  0.00534         0.00508     0.0168        0.00281
    ## 12 C1QC       0.0606   0.0914          0.0915      0.279         0.225  
    ## 13 S100A8     0.388    0.608           0.290       0.502         0.249  
    ## 14 HLA-DQA1   0.510    1.46            1.46        0.548         0.0873 
    ## # … with 10 more variables: melanocyte <dbl>, fibroblast <dbl>, smc <dbl>,
    ## #   rpechor-pericyte <dbl>, rod_rpe <dbl>, b-cell <dbl>, t-cell <dbl>,
    ## #   macrophage-res <dbl>, macrophage-inflam <dbl>, dendritic <dbl>

First, we will create a function to re-color each cartoon subcomponent
based on gene expression. This function will return a list of (1) our
modified (re-colored) XML object and (2) a custom color legend. This is
a somewhat long function, but I have documented each component below.

``` r
generate_cartoon_data <- function(gene,
                                  xml_object, 
                                  expression_matrix,
                                  color_celltype_dictionary,
                                  color = "red"){
  # gene: character string of the gene name for expression visualization
  # xml_object: grImport Picture Object
  # expression_matrix: a dataframe with gene expression information. Rows = genes, 
  #                    columns  = cell types. Gene names must be stored in a column
  #                    named 'gene_name'
  # color_celltype_dictionary: a dictionary (like generated in part 2) that maps
  #                            every colorin the xml_object to a celltype
  # color: character string that indicates the color used for the expression visualization
  

  ### (1) Filter the expression matrix for your gene 
  expression_df <- expression_matrix %>%
    dplyr::filter(gene_name == gene) %>%
    tibble::column_to_rownames("gene_name") %>%
    t() %>%
    as.data.frame() %>%
    tibble::rownames_to_column("celltype")
  
  colnames(expression_df)[2] <- "expression"
  
  ### (2) Find the upper limit of gene expression for your scale
  max_expression <- max(expression_df[,2])
  max_expression <- max_expression + 1  # adding 1 to  max expression makes visualizations
                                        # for lowly expressed genes more accurate
  
  ### (3) Create a blended color scale. Grey = low expression. 
  ###    The user-defined variable `color` = high expression
  colfunc <- grDevices::colorRampPalette(c("grey", color))
  my_colors <- colfunc(101)
  
  expression_df <- expression_df %>%
    dplyr::mutate(percent = round(expression * 100 / max_expression)) %>%
    dplyr::mutate(color = my_colors[as.integer(percent + 1)])
  
  ### (4) Creating a legend of expression
  expression_sequenced <- data.frame(
    expression = seq(from = 0,
                     to = max_expression,
                     by = max_expression/100),
    my_colors)
  
  heatmap_legend <- ggplot2::ggplot(expression_sequenced) +
    ggplot2::geom_tile(aes(x = 1, y = expression, fill = expression)) +
    ggplot2::scale_fill_gradient(low = my_colors[1],
                        high = my_colors[101]) +
    ggplot2::scale_x_continuous(limits=c(0,2),breaks=1, labels = "expression")
  
  leg <- ggpubr::get_legend(heatmap_legend)
  
  
  ### (5) Generate cell type indexes (as shown in Part 3, above)
  celltype_indexes <- vector(mode = "list", length = nrow(color_celltype_dictionary))
  names(celltype_indexes) <- color_celltype_dictionary[["celltype"]]
  
  ## This for loop will fill in the indexes for each celltype
  for(i in 1:length(celltype_indexes)){
    index_vector <- match_colors_and_indexes(color_celltype_tibble = color_celltype_dictionary,
                                             celltype = color_celltype_dictionary[["celltype"]][i],
                                             all_colors = all_colors)
    celltype_indexes[[i]] <- index_vector
  }

  
  ### (6) Re-color the object based on the EXPRESSION color scale
  # (re-coloring modifies the original XML object)
  for(i in 1:nrow(expression_df)){
    celltype <- expression_df[["celltype"]][i]      ## grab our celltype
    indexes <- unlist(celltype_indexes[[celltype]]) ## identify the indexes of this celltype
    color <- expression_df[["color"]][i]            ## grab the color based on the scale in (3)
    
    if(length(indexes) > 0){
      for(p in 1:length(indexes)){                  ## for each index, replace the @rgb colot
          xml_object@paths[indexes[p]]$path@rgb  <- color
      }
    }
  }
  
  return(list(xml_object, leg))  ## return the modified XML object and the custom legend

}
```

### Defining a wrapper function for plotting

Finally, we will create a simple function to plot our cartoon. This
function requires an input ‘cartoon_data’ - a list which is generated by
the above function!

``` r
plot_cartoon <- function(cartoon_data){
  cowplot::plot_grid(
    grImport::pictureGrob(cartoon_data[[1]]),
    cartoon_data[[2]],
    ncol = 2,
    rel_widths = c(0.8, 0.2)
  )
}
```

## Part 5: Let’s make some Cartoon Plots!

First, we visualize expression of *CA4*, which localizes to
choriocapillaris endothelial cells.

``` r
generate_cartoon_data(
  gene = "CA4",
  xml_object = my_cartoon,
  expression_matrix = avg_exp,
  color_celltype_dictionary = color_celltype_tibble,
  color = "red"
) %>%
  plot_cartoon()
```

![](Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-13-1.png)

We can change the color of the expression scale to green simply by
modifying the `color` argument in `generatate_cartoon_data`

``` r
generate_cartoon_data(
  gene = "CA4",
  xml_object = my_cartoon,
  expression_matrix = avg_exp,
  color_celltype_dictionary = color_celltype_tibble,
  color = "green"
) %>%
  plot_cartoon()
```

![](Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-14-1.png)

Finally, let’s visualize the expression of the melanocyte-specific gene,
*MLANA*

``` r
generate_cartoon_data(
  gene = "MLANA",
  xml_object = my_cartoon,
  expression_matrix = avg_exp,
  color_celltype_dictionary = color_celltype_tibble,
  color = "red"
) %>%
  plot_cartoon()
```

![](Retina_Cartoon_vignette_files/figure-markdown_github/unnamed-chunk-15-1.png)

I hope this vignette helps you interact with hand-drawn cartoons in R!
