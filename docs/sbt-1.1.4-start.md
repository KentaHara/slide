# sbtは何故起動するのか

---

## 背景

* sbtのlibraryを読んでいる中で、何が読まれてどんな動作でsbtが立ち上がるのかわからなくなったため

---

## `which sbt`

* コマンドの所在
    ```bash
    $ which sbt
    /usr/local/bin/sbt
    ```

---

## `cat /usr/local/bin/sbt` 

* コマンド内部
    ```bash
    $ cat /usr/local/bin/sbt
    #!/bin/sh
    if [ -f "$HOME/.sbtconfig" ]; then
        echo "Use of ~/.sbtconfig is deprecated, please migrate global settings to /usr/local/etc/sbtopts" >&2
        . "$HOME/.sbtconfig"
    fi
    exec "/usr/local/Cellar/sbt/1.1.4/libexec/bin/sbt" "$@"
    ```
    * `$HOME/.sbtconfig` が存在する場合は、コマンド呼び出し時にconfigを実行する
    * `exec` コマンドを実行して終了する
    * `$@` 引数全てを渡す（今回の場合はsbtコマンドに対しての引数がそのまま渡される）

---

## `cat sbt` 

* dir
    * `/usr/local/Cellar/sbt/1.1.4/libexec/bin/`

--

## function

* `realpath()`
    * 対象とするファイルを変数 `TARGET_FILE` と個数 `COUNT` に入れる
    * `echo "$(pwd -P)/$TARGET_FILE"` 
* `is_cygwin()`
    * Cygwinであれば0、それ以外1を返す
* `cygwinpath()`
    * Cygwin特有のファイルpathをechoする

--
## function

* `usage()`
    * 使用方法等
    * どこか他でまとめたい
* `process_my_args()`
    * parameter指定によるjavaに対する変数宣言等を実施
        * どこか他でまとめたい
    * `[[ "${sbt_version}XXX" != "XXX" ]] && addJava "-Dsbt.version=$sbt_version"`
* `loadConfigFile()`
    * evalしている

--

## libraryの読みこみ


```
. "$(dirname "$(realpath "$0")")/sbt-launch-lib.bash"
```


--

## declare

```
declare -r noshare_opts="-Dsbt.global.base=project/.sbtboot -Dsbt.boot.directory=project/.boot -Dsbt.ivy.home=project/.ivy"
declare -r sbt_opts_file=".sbtopts"
declare -r etc_sbt_opts_file="/usr/local/etc/sbtopts"
declare -r dist_sbt_opts_file="${sbt_home}/conf/sbtopts"
declare -r win_sbt_opts_file="${sbt_home}/conf/sbtconfig.txt"
```

--

## bash下部

```bash
# Here we pull in the default settings configuration.
[[ -f "$dist_sbt_opts_file" ]] && set -- $(loadConfigFile "$dist_sbt_opts_file") "$@"

# Here we pull in the global settings configuration.
[[ -f "$etc_sbt_opts_file" ]] && set -- $(loadConfigFile "$etc_sbt_opts_file") "$@"

#  Pull in the project-level config file, if it exists.
[[ -f "$sbt_opts_file" ]] && set -- $(loadConfigFile "$sbt_opts_file") "$@"

#  Pull in the project-level java config, if it exists.
[[ -f ".jvmopts" ]] && export JAVA_OPTS="$JAVA_OPTS $(loadConfigFile .jvmopts)"

run "$@"
```
    * what's a `run` command
        * `sbt-launch-lib.bash` にて定義されている

---

## cat `sbt-launch-lib.bash`

諸々が動いている定義的なサムシング

* dir
    * `/usr/local/Cellar/sbt/1.1.4/libexec/bin/`


---

## `sbt-launch.jar`

* dir
    * `/usr/local/Cellar/sbt/1.1.4/libexec/bin/`

--

## jarの展開

`jar -xvf sbt-launch.jar`

--

## module properties

```
$ cat module.properties
version=2.3.0-sbt-b18f59ea3bc914a297bb6f1a4f7fb0ace399e310
```
