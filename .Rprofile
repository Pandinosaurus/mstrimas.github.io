library(knitr)
options(stringsAsFactors = FALSE)
options(knitr.table.format = 'markdown')

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
    #fig.width=8.5,
    #fig.height=5.25,
    dev='svg',
    comment='#>',
    tidy=FALSE,
    cache=FALSE,
    error=TRUE,
    warning=TRUE,
    message=FALSE,
    tidy=FALSE)
  
  if (source_rmd == '') {
    if (clean) {
      list.files(fig_path) %>% 
        unlink
    }
    files_rmd <- data.frame(rmd = list.files(
      path = rmd_path,
      full.names = TRUE,
      pattern = '\\.rmd$',
      ignore.case = TRUE,
      recursive = FALSE))
    blah <- files_rmd %>% 
      dplyr::mutate(
        base_name = gsub('\\.rmd$', '', basename(rmd)),
        md = file.path(posts_path, paste0(base_name, '.md')),
        fig_path = file.path(fig_url, paste0(base_name, '_')),
        md_exists = file.exists(md),
        md_render = ifelse(md_exists, overwrite, TRUE)) %>% 
      dplyr::filter(md_render) %>% 
      dplyr::select(rmd, md, fig_path) %>% 
      plyr::a_ply(1, transform, knit_fig_path(
        input = rmd, output = md, fig_path = fig_path, quiet = TRUE))
  } else {
    source_rmd <- file.path(rmd_path, source_rmd)
    stopifnot(file.exists(source_rmd))
    base_name <- gsub('\\.rmd$', '', basename(source_rmd))
    md_file <- file.path(posts_path, paste0(base_name, '.md'))
    fig_path <- file.path(fig_url, paste0(base_name, '_'))
    knit_fig_path(source_rmd, md_file, fig_path = fig_path, quiet = TRUE)
  }
  invisible()
}

knit_fig_path <- function(input, output, fig_path = 'figures/', ...) {
  knitr::opts_chunk$set(fig.path=fig_path)
  knitr::knit(input = input, output = output, ...)
  invisible()
}

# improved list of objects
.ls.objects <- function (pos = 1, pattern, order.by,
                         decreasing=FALSE, head=FALSE, n=5) {
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

# shorthand
lsos <- function(..., n=10) {
  .ls.objects(..., order.by="Size", decreasing=TRUE, head=TRUE, n=n)
}
