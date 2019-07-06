# Alpine Linux shell-fu commands

# Display all packages along with size, sorted by size, largest last

    apk info | xargs -n 1 apk info -s | paste -sd'  \n' | awk '{ print $4 " " $1'} | sort -n
