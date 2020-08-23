all:
	ls *.txt | xargs -I{} bash -c "echo {} | sed 's/\.[^\.]*/$//'" |  xargs -I{} ditaa -S -E {}.txt {}.png
