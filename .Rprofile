library(knitr)
options(stringsAsFactors = FALSE)
options(knitr.table.format = 'markdown')

# From: http://chepec.se/2014/07/16/knitr-jekyll.html

knit_post <- function(source_rmd = "", overwrite = FALSE) {
  # local directory of jekyll site
  site_path <- '/Users/matt/Documents/mstrimas.github.com/'
  # rmd directory (relative to base)
  rmd_path <- file.path(site_path, "_source")
  # relative directory of figures
  fig_url <- "figures/"
  # directory for markdown output
  posts_path <- paste0(site_path, "_posts")
  
  # knitr options
  knitr::render_jekyll(highlight = "pygments")
  knitr::opts_knit$set(
    base.url="/",
    base.dir=site_path)
  knitr::opts_chunk$set(
    fig.path=fig_url,
    fig.width=8.5,
    fig.height=5.25,
    dev='svg',
    cache=FALSE,
    error=TRUE,
    warning=TRUE,
    message=FALSE,
    tidy=FALSE)
  
  if (source_rmd == "") {
    files_rmd <- data.frame(rmd = list.files(
      path = rmd_path,
      full.names = TRUE,
      pattern = "\\.rmd$",
      ignore.case = TRUE,
      recursive = FALSE))
    blah <- files_rmd %>% 
      dplyr::mutate(
        base_name = gsub("\\.rmd$", "", basename(rmd)),
        md = file.path(posts_path, paste0(base_name, ".md")),
        fig_path = file.path(fig_url, paste0(base_name, "_")),
        md_exists = file.exists(md),
        md_render = ifelse(md_exists, overwrite, TRUE)) %>% 
      dplyr::filter(md_render) %>% 
      dplyr::select(rmd, md, fig_path) %>% 
      plyr::a_ply(1, transform, knit_fig_path(
        input = rmd, output = md, fig_path = fig_path, quiet = TRUE))
  } else {
    source_rmd <- file.path(rmd_path, source_rmd)
    stopifnot(file.exists(source_rmd))
    base_name <- gsub("\\.rmd$", "", basename(source_rmd))
    md_file <- file.path(posts_path, paste0(base_name, ".md"))
    fig_path <- file.path(fig_url, paste0(base_name, "_"))
    knit_fig_path(source_rmd, md_file, fig_path = fig_path, quiet = TRUE)
  }
  invisible()
}

knit_fig_path <- function(input, output, fig_path = 'figures/', ...) {
  knitr::opts_chunk$set(fig.path=fig_path)
  knitr::knit(input = input, output = output, ...)
  invisible()
}