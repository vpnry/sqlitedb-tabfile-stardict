Dear the later version of me, 

Those are the abridged script files that you used to convert **SQLite database** files to:

```
1. Tab file
2. Javascript object file
3. Stardict dictionary file
```

> Tip:
>
> + We are using Ubuntu.
> + In `Terminal`, the bash files should be `chmod +x yourbashfile` before running it `./yourbashfile`
> + If you want to run all of these files, your machine should have:
>
> ```
> # must have: 
> python3, sqlite3, mawk (faster than the GNU awk?)
> 
> # if want to conver to JS object, stardict dictionary: 
> nodejs, pyglossary
> ```

# 1. Convert SQLite3 database files to tab files:

Install `sqlite3, mawk` with these commands:

```bash
sudo apt update
sudo apt install mawk sqlite sqlite-doc

```

In the below script `-separator $'\t'` is using a tab as it separator, you can change it to any chars your need.

```bash
#!/bin/bash

# inPutDBDir is your SQLite database files folder
# tabfiles_done is the output folder
# dictionary is the table name

COUNTER=0
NEWDIR="tabfiles_done/"
mkdir -p $NEWDIR
for db in inPutDBDir/*; do
  tsvname=$(echo $db | sed -e "s/\.db/\.tsv/g" | mawk -v FS="/" '{ print $NF}')
  sqlite3 -noheader -separator $'\t' $db "select * from dictionary" >$NEWDIR$tsvname
  COUNTER=$((COUNTER + 1))
  echo $COUNTER". "$NEWDIR$tsvname
done
echo "Raw tab file done: "$COUNTER" files"
```

# 2. To fix duplicate entries in the tab files

In the original SQLite database files, they allow duplicated entries as they have different IDs. 

However if you use `startdict-editor` tool to convert a tab file with duplicated entries to Stardict dictionary files, it will yield errors. 

And if you convert it to a `js object`, the JS object will use only the latest definition in the object. For example:

```javascript
let myDictionary = {
  human: "happy",
  human: "mind",
  tusita: "deva"
}
myDictionary["human"] // mind
```

This bash script will concatenate the definitions. So the later JS object should be something like:

```javascript
let myDictionary = {
  human: "happy YOURDEFIDIVIDER mind",
  tusita: "deva"
}
```
 *YOURDEFIDIVIDER* is the divider that you can put in the script file, we just use a blank space.


```bash
#!/bin/bash

# chmod 777 fixdup; ./fixdup;
# If I name this file as fixdup.sh and run sh fixdup.sh, the $'\t' does not work!?
# To use as tab file for Stardict, you need to fix dublicate entry

# YOURDEFIDIVIDER: I just simply use a blank space for it, you may also use <br> etc as divider

COUNTER=0
SAVEDIR="tabfiles_ok/"
mkdir -p $SAVEDIR
for db in tabfiles_done/*; do
  fname=$(echo $db | mawk -v FS="/" '{ print $NF}')
  cat $db | mawk -v FS="\t" '{
  if (words[$1]) words[$1]= words[$1] "; YOURDEFIDIVIDER " $2 
  else words[$1] = $2
} END {
  for(w in words)
    print w "\t" words[w]
}' | sort | grep "\S" >>$SAVEDIR$fname
  COUNTER=$((COUNTER + 1))
  echo $COUNTER". "$SAVEDIR$fname
done

echo "Done: " $COUNTER
echo ""

```

# 3. From tab files (removed duplicated entries) to JavaScript dictionary object

Run this with Nodejs (on Ubuntu I installed Nodejs via `nvm`). You can manually provide `filePaths` or write a  function to auto do it, for [example](https://stackoverflow.com/a/63111390). To manually list the files:

```bash
ls tabfiles_ok > files.txt
```

And then use your Reg-ex skills to put it in the array manually.

```javascript
const fs = require("fs");
let filePaths = ["tabfiles_ok/f-na-me1.tsv", "tabfiles_ok/f-na-me2.tsv"];

function toJS() {
  console.log("Converting to JS dictionary files");
  let i = 0,
    lenP = filePaths.length;
  console.log("Total files: " + lenP);
  for (let f of filePaths) {
    let fc = fs.readFileSync(f, { encoding: "utf-8", flag: "r" });
    let r = f.split("/");
    let name = r[r.length - 1].trim();
    name = name.substring(0, name.indexOf("."));
    let varn = name.replace(/-/g, "");
    let str = "const " + varn + " = {\n";
    fc = fc.split("\n");
    for (let line of fc) {
      line = line.trim();
      let mar = line.indexOf("\t");
      let word = line.substring(0, mar);
      word = word.trim();
      let defi = line.substring(mar);
      defi = defi.replace(/'/g, "\\'");
      defi = defi.trim();
      str += "'" + word + "':'" + defi + "',\n";
    }

    str += '"sadhusadhusadhu":`sadhusadhusadhu!`\n};\n';
    str += `module.exports = ${varn};`;

    fs.writeFileSync(name + ".js", str);
    i++;
    console.log(i + ". Converted: " + f);
    // if (i == 2) break;
  }

  console.log("");
  console.log("Done! Processed: " + i + "/" + lenP + " files");
}

```

# 4. To convert tab files to Startdict with pyglossary

> + If you only have a few files to convert to dictionary files, simply use `stardict-editor` to do the task. 
> + On Ubuntu 20.04, `stardict-editor`, running on `wine`, crashed and failed to convert a big tab file to Stardict format . maybe my tab file is too big)

+ Install `pyglossary` through `pip3 install pyglossary` alone is not enough for you to run it, you need to clone its project too:

```bash
  pip3 install pyglossary
  
  git clone https://github.com/ilius/pyglossary.git
  cd pyglossary/
  
  # To run with UI (if you have instaled it)
  python3 pyglossary.pyw --ui=gtk
  
  # To show the help
  python3 pyglossary.pyw --help

  
```

+ If you want to run with command, in Pyglossary project directory that you have just cloned, create a `inputTabFileDir` folder and copy all of your tab files in to it.

+ We will also create a file  `myBashScript` with this contents:

  ```bash
  #!/bin/bash
  
  COUNTER=0
  SAVEDIR="stardictDone/"
  mkdir -p $SAVEDIR
  for db in inputTabFileDir/*; do
    ifoname=$(echo $db | sed -e "s/\.tsv/\.ifo/g" | mawk -v FS="/" '{ print $NF}')
    python3 pyglossary.pyw $db $SAVEDIR$ifoname -v3 --read-format=Tabfile --write-format=Stardict --write-options=sametypesequence=h
    COUNTER=$((COUNTER + 1))
    echo $COUNTER". "$SAVEDIR$ifoname
  done
  echo "Converted : "$COUNTER" files"
  
  ```

The main command is this:

```bash
python3 pyglossary.pyw INPUTFILE OUTPUTFILE -v3 --read-format=Tabfile --write-format=Stardict --write-options=sametypesequence=h
```

`sametypesequence=h`: the definitions type is HTML, see StarDictFileFormat link below to read more. 


  So we have this structure, something like:

  ```
  .
  ├── about
  ├── main.py
  ├── pyglossary.pyw
  ................
  ├── myBashScript
  ├── inputTabFileDir
  ├── stardictDone
  ................
  ```

  Now you can run:

```bash
chmod +x myBashScript
./myBashScript
```

You can use the Stardict dictionary files with **Golden Dict ** on Ubuntu, **Dicty** on iOS, **ColorDict** on Android and so on.

**Related readings:** 

+ PyGlossary: https://github.com/ilius/pyglossary
+ StarDictFileFormat https://raw.githubusercontent.com/huzheng001/stardict-3/master/dict/doc/StarDictFileFormat



May you all be well and happy!
