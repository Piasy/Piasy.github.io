
``` bash
# inspired by http://stackoverflow.com/a/4252503/3077508

# install fonts
mkdir -p ~/.fonts/Hack
wget https://github.com/chrissimpkins/Hack/releases/download/v2.020/Hack-v2_020-ttf.zip
unzip Hack-v2_020-ttf.zip ~/.fonts/Hack
# and install msyh
fc-cache -f -v

# install softwares
sudo apt-get update
sudo apt-get install -y highlight
sudo apt-get install -y ghostscript
sudo apt-get install -y xvfb
sudo apt-get install -y imagemagick
wget http://download.gna.org/wkhtmltopdf/0.12/0.12.3/wkhtmltox-0.12.3_linux-generic-amd64.tar.xz
tar xvf wkhtmltox-0.12.3_linux-generic-amd64.tar.xz

# input is 1.java
highlight -i 1.java -o 1.html --include-style --line-numbers --font Hack,\'msyh\'
wkhtmltox/bin/wkhtmltopdf --quiet --dpi 1200 1.html 1.ps
gs -q -dBATCH -dNOPAUSE -dSAFER -dNOPROMPT -sDEVICE=png16m -dDEVICEXRESOLUTION=600 -dDEVICEYRESOLUTION=600 -dDEVICEWIDTH=4958 -dDEVICEHEIGHT=7017 -dNOPLATFONTS -dTextAlphaBits=4 -sOutputFile=1.png 1.ps
convert -trim +repage -trim +repage -bordercolor "#f0f0f0" -border 25x25 1.png 1.trim.png

# output is 1.trim.png
```