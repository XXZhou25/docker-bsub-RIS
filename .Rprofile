cat <<EOF > $HOME/.Rprofile
vals <- paste('/storage1/fs1/yeli/Active/R_caches/danting/',paste(R.version$major,R.version$minor,sep="."),sep="")
for (devlib in vals) {
    if (!file.exists(devlib))
    dir.create(devlib)
x <- .libPaths()
x <- .libPaths(c(devlib,x))
}
rm(x,vals)
EOF
