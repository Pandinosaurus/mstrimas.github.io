if (file.exists('~/.Rprofile')) {
  source('~/.Rprofile')
}

library(knitr)
library(magrittr)
#options(stringsAsFactors = FALSE)
options(knitr.table.format = 'markdown')

clean_post <- function(source_rmd, fig_path) {
  gsub('\\.rmd$', '', basename(source_rmd)) %>% 
    list.files(path = fig_path, pattern = ., full.names = TRUE) %>% 
    unlink
}

is_published <- function(source_rmd) {
  published <- readLines(source_rmd, warn = FALSE, n = 25) %>% 
    tolower %>% 
    grep("published", ., value = TRUE) %>% 
    sapply(function(x) grepl('true', x, fixed = TRUE))
  if (length(published) > 0 && !published[[1]]) {
    return(FALSE)
  } else {
    return(TRUE)
  }
}

knit_fig_path <- function(input, output, fig_path = 'figures/', ...) {
  knitr::opts_chunk$set(fig.path=fig_path)
  knitr::knit(input = input, output = output, ...)
  invisible()
}

# From: http://chepec.se/2014/07/16/knitr-jekyll.html
knit_post <- function(source_rmd = '', overwrite = FALSE, clean = TRUE) {
  # local directory of jekyll site
  site_path <- '/Users/matt/Documents/mstrimas.github.com/'
  # rmd directory (relative to base)
  rmd_path <- file.path(site_path, '_source/')
  # relative directory of figures
  fig_url <- 'figures/'
  # directory for markdown output
  posts_path <- file.path(site_path, '_posts/')
  # knitr cache directory
  cache_path <- file.path(rmd_path, 'cache/')
  
  # knitr options
  #knitr::render_jekyll(highlight = 'pygments')
  knitr::render_markdown(strict = F)
  knitr::opts_knit$set(
    base.url='/',
    base.dir=site_path)
  knitr::opts_chunk$set(
    cache.path=cache_path,
    fig.path=fig_url,
    fig.align='center',
    dpi=96,
    fig.width=480/96,
    fig.height=480/96,
    dev='svg',
    comment='##',
    tidy=FALSE,
    cache=FALSE,
    error=TRUE,
    warning=TRUE,
    message=FALSE,
    collapse=FALSE)
  
  if (source_rmd == '') {
    files_rmd <- dplyr::data_frame(rmd = list.files(
      path = rmd_path,
      full.names = TRUE,
      pattern = '\\.rmd$',
      ignore.case = TRUE,
      recursive = FALSE))
    # create list of files to knit
    file_df <- files_rmd %>% 
      #dplyr::group_by(rmd) %>% 
      dplyr::mutate(
        base_name = gsub('\\.rmd$', '', basename(rmd)),
        md = file.path(posts_path, paste0(base_name, '.md')),
        fig_path = file.path(fig_url, paste0(base_name, '_')),
        md_exists = file.exists(md),
        published = sapply(rmd, is_published),
        md_render = ifelse(published & (overwrite | !md_exists), TRUE, FALSE)) %>%
      dplyr::filter(md_render)
    
    if (nrow(file_df) == 0) {
      return(invisible())
    }
    
    # clean
    if (clean) {
      file_df %>% 
        dplyr::select(rmd) %>% 
        plyr::a_ply(1, transform, clean_post(
          source_rmd = basename(rmd), fig_path = file.path(site_path, fig_url)))
    }
    
    # knit
    file_df %>% 
      dplyr::select(rmd, md, fig_path) %>% 
      plyr::a_ply(1, transform, knit_fig_path(
        input = rmd, output = md, fig_path = fig_path, quiet = TRUE))
  } else {
    source_rmd <- file.path(rmd_path, source_rmd)
    stopifnot(file.exists(source_rmd))
    base_name <- gsub('\\.rmd$', '', basename(source_rmd))
    md_file <- file.path(posts_path, paste0(base_name, '.md'))
    fig_path <- file.path(fig_url, paste0(base_name, '_'))
    clean_post(source_rmd = basename(source_rmd), fig_path = file.path(site_path, fig_url))
    knit_fig_path(source_rmd, md_file, fig_path = fig_path, quiet = TRUE)
  }
  invisible()
}

# improved list of objects
lsos <- function (pos = 1, pattern, order.by = "size",
                         decreasing = TRUE, head = TRUE, n = 10) {
  napply <- function(names, fn) sapply(names, function(x)
    fn(get(x, pos = pos)))
  names <- ls(pos = pos, pattern = pattern)
  obj.class <- napply(names, function(x) as.character(class(x))[1])
  obj.mode <- napply(names, mode)
  obj.type <- ifelse(is.na(obj.class), obj.mode, obj.class)
  obj.prettysize <- napply(names, function(x) {
    capture.output(format(utils::object.size(x), units = "auto")) })
  obj.size <- napply(names, object.size)
  obj.dim <- t(napply(names, function(x)
    as.numeric(dim(x))[1:2]))
  vec <- is.na(obj.dim)[, 1] & (obj.type != "function")
  obj.dim[vec, 1] <- napply(names, length)[vec]
  out <- data.frame(obj.type, obj.size, obj.prettysize, obj.dim)
  names(out) <- c("Type", "Size", "PrettySize", "Rows", "Columns")
  if (!missing(order.by))
    out <- out[order(out[[order.by]], decreasing=decreasing), ]
  if (head)
    out <- head(out, n)
  out
}
