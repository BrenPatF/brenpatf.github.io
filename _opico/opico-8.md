---
layout: post
title:  "OPICO 8: Automation"
date:   2024-08-18 06:00:00 +0100
tags:   ["oracle", "optimization", "combination", "permutation", "recursion", "iteration", "knapsack", "sql", "testing", "verification", "automation"]
opico_prev: /opico/opico-7/
---
<link rel="stylesheet" type="text/css" href="/assets/css/styles.css">
#### Part 8 in a series on: Optimization Problems with Items and Categories in Oracle

<div class="opico-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 OPICO Series:</strong>
  <a href="/2024/06/30/opico-series-index.html">Index</a>
  {% if page.opico_prev %}
    | <a href="{{ page.opico_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.opico_next %}
    | <a href="{{ page.opico_next }}">Next ▶</a>
  {% endif %}
</div>

In the seventh article, we discussed verification of the solution methods developed.

In the current article, we describe the various kinds of automation used in the development, testing and installation of the code and other artefacts for this series of articles.

We conclude by listing some of the features and concepts considered in the whole series.

<img src="/images/2024/08/18/P1040951.JPG" style="width: 100%; max-width: 100%;" /><br />
[Image by me, Zermatt 2016]
# Contents
[&darr; 1 Installation](#1-installation)<br />
[&darr; 2 Running the Solution Methods](#2-running-the-solution-methods)<br />
[&darr; 3 Code Instrumentation](#3-code-instrumentation)<br />
[&darr; 4 Unit Testing](#4-unit-testing)<br />
[&darr; 5 Blog Internal Links Hierarchy](#5-blog-internal-links-hierarchy)<br />
[&darr; 6 Conclusion](#6-conclusion)<br />

There has been a trend in recent years to automate more and more aspects of software development that previously would have been manual, with growing recognition of the many benefits that come with automation.

We now discuss the multiple ways in which we use code for process automation in this series of articles, and in the associated GitHub project.

## 1 Installation
[&uarr; Contents](#contents)<br />
[&darr; Multi-Step Installation](#multi-step-installation)<br />
[&darr; Automated Installation](#automated-installation)<br />

All the scripts, database objects and datasets needed to replicate the results from this series of articles are contained in a GitHub project [Optimization Problems with Items and Categories in Oracle](https://github.com/BrenPatF/item_category_optimization_oracle).

The Oracle installation can be performed via a single powershell script, or in a series of smaller steps.

### Multi-Step Installation
[&uarr; 1 Installation](#1-installation)<br />
[&darr; File System Installs](#file-system-installs)<br />
[&darr; Database Installs](#database-installs)<br />

#### File System Installs
[&uarr; Multi-Step Installation](#multi-step-installation)<br />

- Copy the following files to the server folder pointed to by the Oracle directory INPUT_DIR:

    - fantasy_premier_league_player_stats.csv
    - unit_test/tt_item_cat_seqs.purely_wrap_best_combis_inp.json

The file system installation can also be performed, assuming C:\input as INPUT_DIR, simply by running the following powershell script, `Copy-DataFilesInput.ps1`:

```powershell
$ ./Copy-DataFilesInput
```

#### Database Installs
[&uarr; Multi-Step Installation](#multi-step-installation)<br />

The Oracle database installation is implemented through a small number of driver scripts: One per Oracle schema and folder, separating out the prerequisite installs from the main ones, and base from unit test components.

| Script              | Schema | Folder             | Purpose                                         |
|:--------------------|:-------|:-------------------|:------------------------------------------------|
| install_sys.sql     | sys    | install_prereq     | Create lib and app schemas and Oracle directory |
| install_lib_all.sql | lib    | install_prereq\lib | Install lib components                          |
| c_syns_all.sql      | app    | install_prereq\app | Create app synonyms to lib                      |
| install_ico.sql     | app    | app                | Install app base components                     |
| install_ico_tt.sql  | app    | app                | Install app unit test components                |

### Automated Installation
[&uarr; 1 Installation](#1-installation)<br />
[&darr; One Step Install](#one-step-install)<br />
[&darr; Powershell Script Install-Ico.ps1](#powershell-script-install-icops1)<br />

Although the manual installation is simplified by having a small number of driver scripts, and takes only a minute or two to perform, it seemed desirable to have a fully automated option.

#### One Step Install
[&uarr; Automated Installation](#automated-installation)<br />

The installation can be performed simply by running the following powershell script, `Install-Ico.ps1`:

```powershell
./Install-Ico
```

#### Powershell Script Install-Ico.ps1
[&uarr; Automated Installation](#automated-installation)<br />

```powershell
. ./Copy-DataFilesInput
if (-not $IsFolder) {
    "Could not copy files, so aborting main script..."
    exit
}
$installs = @(@{folder = 'install_prereq'; script = 'drop_utils_users'; schema = 'sys'},
              @{folder = 'install_prereq'; script = 'install_sys'; schema = 'sys'},
              @{folder = 'install_prereq\lib'; script = 'install_lib_all'; schema = 'lib'},
              @{folder = 'install_prereq\app'; script = 'c_syns_all'; schema = 'app'},
              @{folder = 'app'; script = 'install_ico'; schema = 'app'},
              @{folder = 'app'; script = 'install_ico_tt'; schema = 'app'})
$installs

Foreach($i in $installs){
    sl ($PSScriptRoot + '/' + $i.folder)
    $script = '@./' + $i.script
    $sysdba = ''
    if ($i.schema -eq 'sys') {
        $sysdba = ' AS SYSDBA'
    }
    $conn = $i.schema + '/' + $i.schema + '@orclpdb' + $sysdba
    'Executing: ' + $script + ' for connection ' + $conn
    & sqlplus $conn $script
}
sl $PSScriptRoot
```

The script starts by executing the file-copying script in the current shell, with a check that the INPUT_DIR folder exists, and aborting if not.

The database installs are then data-driven, running a loop over a powershell array of hashtables. At each iteration we navigate to the specified folder, open a sqplus session for the specified schema and execute the specified script.

##### Powershell Script Copy-DataFilesInput.ps1

The script executed in Install-Ico.ps1 is:

```powershell
$inputPath = "c:/input"
$utFile = "./unit_test/tt_item_cat_seqs.purely_wrap_best_combis_inp.json"
$csvFile = "./fantasy_premier_league_player_stats.csv"

$IsFolder = $true
if (Test-Path $inputPath) {
    if (Test-Path -PathType Container $inputPath) {
        "The item $inputPath is a folder, copy there..."
    } else {
        "The item $inputPath is a file, aborting..."
        $IsFolder = $false
    }
} else {
    "The item $inputPath does not exist, creating folder..."
    New-Item -ItemType Directory -Force -Path $inputPath
}
if ($IsFolder) {
    Copy-Item $utFile $inputPath
    "Copied $utFile to $inputPath"
    Copy-Item $csvFile $inputPath
    "Copied $csvFile to $inputPath"
}
```
The script assumes that the Oracle INPUT_DIR folder is c:/input. It checks whether the item exists as a folder or a file. If it's a file it sets a flag to abort the calling script, and if it does not exist it creates it as a folder. If the item was not a file it copies the unit test input JSON file and the dataset file for the England example to the folder.

## 2 Running the Solution Methods
[&uarr; Contents](#contents)<br />
[&darr; Sqlplus Scripts](#sqlplus-scripts)<br />
[&darr; Powershell Driver Script - Run-All.ps1](#powershell-driver-script---run-allps1)<br />
[&darr; Output Logs](#output-logs)<br />

This series of articles involves running multiple solution methods against three datasets, using several sets of parameters that affect the behaviour and hence the performance of the methods. While it's possible to run each combination of solution method, dataset and parameter set manually, it becomes quite cumbersome and time-consuming once the number of combinations rises to more than a few.

A better approach is to use a scripting language to coordinate the different runs, just as we did in automating the installation steps. However, in this case we need to structure our driver script and the SQL scripts that it calls carefully to meet additional requirements:

- We want log files for each run, with results and instrumentation, that don't overwrite each other
- With a large number of individual detailed log files, we would like to have a summary level log file too
- As run times can vary depending on background processing, we would like to run the driving script multiple times without overwriting previous runs, so we'll use a run index and store results in a subfolder with the index as suffix in the subfolder name and in the summary log name

### Sqlplus Scripts
[&uarr; 2 Running the Solution Methods](#2-running-the-solution-methods)<br />

Each of the solution method scripts has as first parameter, DS, a 3-letter code for the dataset: sml / bra / eng. Before the set of scripts is run in a separate sqlplus session for each dataset, DS, two SQL scripts are run first:

- c_temp_tables.sql recreates the temporary tables and re-compiles the packages
- views_DS.sql points the dataset views to the dataset DS

The DS parameter is then passed in first to the scripts below, and is used as a suffix on the log file name. Any remaining parameters are added as additional suffixes: This way we can identify the dataset and parameter values used just from the log file name, and one run can't overwrite another.

| SQL Script           | Param 2       | Param 3     | Purpose                                                                                |
|:---------------------|:--------------|:------------|:---------------------------------------------------------------------------------------|
| item_seqs            | Sequence type |             | List sequences of given type of size 3, multiple methods                               |
| item_cat_seqs        | KEEP_NUM      | MIN_VALUE   | Run the base PL/SQL driven methods (i.e. excluding the iterative refinement ones)      |
| item_cat_seqs_rsf    | KEEP_NUM      | MIN_VALUE   | Run the recursive subquery factor methods apart from the post-validation approach      |
| item_cat_seqs_pv_rsf |               |             | Run the recursive subquery factor method using the post-validation approach (sml only) |
| item_cat_seqs_loop   | KEEP_START    | KEEP_FACTOR | Run the iterative refinement PL/SQL driven methods                                     |

#### Sqlplus Script Spool File Handling

The SQL scripts define parameters in the following way (for example) with a call to another script, initspool.sql, that executes the spool statement, as well as listing some database parameters, the time etc.

```sql
DEFINE DS = &1
DEFINE KEEP_NUM = &2
DEFINE MIN_VALUE = &3
@..\install_prereq\initspool item_cat_seqs_&DS._&KEEP_NUM._&MIN_VALUE
```

The spool statement in initspool.sql is simply:
```sql
SPOOL &1..log
```

### Powershell Driver Script - Run-All.ps1
[&uarr; 2 Running the Solution Methods](#2-running-the-solution-methods)<br />

```powershell
Date -format "dd-MMM-yy HH:mm:ss"
$startTime = Get-Date
$directories = Get-ChildItem -Directory | Where-Object { $_.Name -match "^results_\d+$" }
[int]$maxIndex = 0
if ($directories.Count -gt 0) {
    [int[]]$indexLis = $directories |
        ForEach-Object {
            $_.Name -replace 'results_', ''
        }
    $maxIndex = ($indexLis | Measure-Object -Maximum).Maximum
}
$nxtIndex = ($maxIndex + 1).ToString("D2")
$newDir = ('results_' + $nxtIndex)
New-Item ('results_' + $nxtIndex) -ItemType Directory
$logFile = $PSScriptRoot + '\Run-All_' + $nxtIndex + '.log'
$ddl = 'c_temp_tables'
$inputs = [ordered]@{
    sml = [ordered]@{views_sml              = @()
                     item_seqs              = @(@('MP'),@('MC'),@('SP'),@('SC'))
                     item_cat_seqs          = @(,@(10, 0))
                     item_cat_seqs_rsf      = @(,@(10, 0))
                     item_cat_seqs_pv_rsf   = @()
                     item_cat_seqs_loop     = @(,@(10, 3))}
    bra = [ordered]@{views_bra              = @()
                     item_cat_seqs          = @(@(10, 0),@(100, 0),@(100, 10748),@(0, 10748))
                     item_cat_seqs_rsf      = @(@(10, 0),@(100, 0),@(100, 10748),@(0, 10748))
                     item_cat_seqs_loop     = @(,@(10, 3))}
    eng = [ordered]@{views_eng              = @()
                     item_cat_seqs          = @(@(50, 0),@(300, 0),@(300, 1952),@(0, 1952))
                     item_cat_seqs_rsf      = @(@(50, 0),@(300, 0),@(300, 1952),@(0, 1952))
                     item_cat_seqs_loop     = @(,@(50, 3))}
}
foreach($i in $inputs.Keys){
    Set-Location $newDir
    $i
    [string]$cmdLis = ('@..\' + $ddl + [Environment]::NewLine)
    $logLis = @()
    foreach($v in $inputs[$i]) {
        foreach($k in $v.Keys) {
            if($v[$k].length -eq 0) {
                $cmdLis += ('@..\' + $k + [Environment]::NewLine)
            }
            foreach($p in $v[$k]) {
                $newCmd = ('@..\' + $k + ' ' + $i + ' ' + $p[0] + ' ' + $p[1])
                ("newCmd = " + $newCmd)
                $p
                $cmdLis += ($newCmd + [Environment]::NewLine)
                if($p[1] -ne $null) {$logLis += ($k + '_' + $i + '_' + $p[0] + '_' + $p[1] + '.log')}
            }
        }
    }
    $cmdLis
    $output = $cmdLis | sqlplus 'app/app@orclpdb'
    Set-Location ..
    foreach($l in $logLis) {
        $f = $newDir + '\' + $l
        $l | Out-File $logFile -Append -encoding utf8
        Get-Content $f | Select-String -Pattern 'Timer Set: Item_Cat_Seqs,' -Context 0, 17 | Out-File $logFile -Append -encoding utf8
        Get-Content $f | Select-String -Pattern 'Timer Set: Item_Cat_Seqs_RSF' -Context 0, 14 | Out-File $logFile -Append -encoding utf8
        Get-Content $f | Select-String -Pattern 'Timer Set: Item_Cat_Seqs_Loop' -Context 0, 12 | Out-File $logFile -Append -encoding utf8
    }
}
$elapsedTime = (Get-Date) - $startTime
$roundedTime = [math]::Round($elapsedTime.TotalSeconds)
"Total time taken: $roundedTime seconds" | Out-File $logFile -Append -encoding utf8
```

#### Summary Level Logging
The run logs contain quite detailed instrumentation such as execution plans and code timing tables for each solution method run. We supplement this by a script level code timing table that lists overall statistics by method. It is also useful to extract the script level tables into a single log file for the powershell script. This is effected by storing the log file names in a powershell array, then looping over each one doing a pattern match on the timer set names:

```powershell
    foreach($l in $logLis) {
        $l
        Get-Content $l | Select-String -Pattern 'Timer Set: Item_Cat_Seqs' -Context 0, 16
    }
```

You can see the script level tables in the Output Logs section below.

#### Data Driven Logic

Rather than have a long sequence of sqlplus executions with hard-coded parameter values, mixed in with additional logic, it seemed a cleaner approach to declare data structures to hold datasets names, script names, and parameter values. Then we can iterate over the data structure, generating and executing the commands, while storing the log file names. This leads to a shorter and more readable script.

The variable $inputs is a hashtable with structure as follows:

| Level | Entity                    | Example                                         |
|:------|:--------------------------|:------------------------------------------------|
| 1     | Dataset code              | bra                                             |
| 2     | SQL Script name           | item_cat_seqs                                   |
| 3     | Array of parameter sets   | @(@(10, 0),@(100, 0),@(100, 10748),@(0, 10748)) |
| 4     | Array of parameter values | @(10, 0)                                        |

### Output Logs
[&uarr; 2 Running the Solution Methods](#2-running-the-solution-methods)<br />
[&darr; Run Instance 3 Summary - Run-All_03.log](#run-instance-3-summary---run-all_03log)<br />
[&darr; Run Instance 3 Results Folder - results_03](#run-instance-3-results-folder---results_03)<br />

#### Run Instance 3 Summary - Run-All_03.log
[&uarr; Output Logs](#output-logs)<br />

<div class="scrollbox">

<pre>
item_cat_seqs_sml_10_0.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:03, written at 23:37:04
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          0.17        0.16           1        0.16800        0.16000
  Solutions by Category                      0.03        0.02           1        0.03400        0.02000
  Pop_Table_Iterate_Base                     0.19        0.14           1        0.18800        0.14000
  Pop_Table_Iterate_Link                     0.24        0.17           1        0.24100        0.17000
  Pop_Table_Iterate_Link - item level        0.03        0.01           1        0.03200        0.01000
  Pop_Table_Iterate_Base_Link                0.21        0.19           1        0.20600        0.19000
  Array_Iterate                              0.12        0.10           1        0.12300        0.10000
  Pop_Table_Recurse                          0.11        0.10           1        0.11200        0.10000
  Array_Recurse                              0.13        0.10           1        0.13000        0.10000
  (Other)                                    0.00        0.00           1        0.00300        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      1.24        0.99          10        0.12370        0.09900
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00990, CPU: 0.00792]


item_cat_seqs_rsf_sml_10_0.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:37:04, written at 23:37:06
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                         0.03        0.01           1        0.02900        0.01000
  rsf_sql_material_v                0.03        0.04           1        0.02700        0.04000
  Item_Cat_Seqs.Init                0.01        0.01           3        0.00433        0.00333
  rsf_irk_irs_tabs_v                0.02        0.03           1        0.02200        0.03000
  rsf_irk_tab_where_fun_ts_v        0.12        0.09           1        0.11800        0.09000
  rsf_irk_tab_where_fun_v           0.02        0.00           1        0.01500        0.00000
  (Other)                           1.32        1.23           1        1.32200        1.23000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                             1.55        1.41           9        0.17178        0.15667
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00962, CPU: 0.00962]


item_cat_seqs_loop_sml_10_3.log

> Timer Set: Item_Cat_Seqs_Loop, Constructed at 10 Jun 2024 23:37:06, written at 23:37:08
  =======================================================================================
  Timer                                       Elapsed         CPU       Calls       Ela/Call       CPU/Call
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Iteratively_Refine_Recurse - path level        0.33        0.31           1        0.32900        0.31000
  Iteratively_Refine_Recurse - item level        0.32        0.32           1        0.32100        0.32000
  Iteratively_Refine_Iterate - path level        0.32        0.28           1        0.31500        0.28000
  Iteratively_Refine_RSF - path level            0.36        0.33           1        0.36300        0.33000
  (Other)                                        0.00        0.00           1        0.00100        0.00000
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                          1.33        1.24           5        0.26580        0.24800
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00934, CPU: 0.00849]


item_cat_seqs_bra_10_0.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:09, written at 23:37:12
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          0.44        0.37           1        0.44100        0.37000
  Solutions by Category                      0.02        0.01           1        0.02000        0.01000
  Pop_Table_Iterate_Base                     0.41        0.41           1        0.40500        0.41000
  Pop_Table_Iterate_Link                     0.42        0.41           1        0.41700        0.41000
  Pop_Table_Iterate_Link - item level        0.02        0.00           1        0.02100        0.00000
  Pop_Table_Iterate_Base_Link                0.42        0.38           1        0.42000        0.38000
  Array_Iterate                              0.39        0.37           1        0.38800        0.37000
  Pop_Table_Recurse                          0.36        0.35           1        0.35800        0.35000
  Array_Recurse                              0.39        0.36           1        0.38600        0.36000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      2.86        2.66          10        0.28570        0.26600
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00909, CPU: 0.00636]


item_cat_seqs_bra_100_0.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:12, written at 23:37:16
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          0.59        0.52           1        0.59200        0.52000
  Solutions by Category                      0.01        0.00           1        0.00700        0.00000
  Pop_Table_Iterate_Base                     0.55        0.51           1        0.55400        0.51000
  Pop_Table_Iterate_Link                     0.55        0.54           1        0.55100        0.54000
  Pop_Table_Iterate_Link - item level        0.01        0.00           1        0.01300        0.00000
  Pop_Table_Iterate_Base_Link                0.51        0.49           1        0.51000        0.49000
  Array_Iterate                              0.62        0.58           1        0.61800        0.58000
  Pop_Table_Recurse                          0.56        0.53           1        0.56300        0.53000
  Array_Recurse                              0.61        0.59           1        0.61300        0.59000
  (Other)                                    0.00        0.01           1        0.00100        0.01000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      4.02        3.77          10        0.40220        0.37700
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00877, CPU: 0.00877]


item_cat_seqs_bra_100_10748.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:16, written at 23:37:19
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          0.36        0.36           1        0.35500        0.36000
  Solutions by Category                      0.01        0.00           1        0.00600        0.00000
  Pop_Table_Iterate_Base                     0.35        0.34           1        0.35200        0.34000
  Pop_Table_Iterate_Link                     0.36        0.35           1        0.35900        0.35000
  Pop_Table_Iterate_Link - item level        0.01        0.00           1        0.01100        0.00000
  Pop_Table_Iterate_Base_Link                0.36        0.32           1        0.35900        0.32000
  Array_Iterate                              0.37        0.33           1        0.36600        0.33000
  Pop_Table_Recurse                          0.36        0.36           1        0.35800        0.36000
  Array_Recurse                              0.38        0.31           1        0.38300        0.31000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      2.55        2.37          10        0.25500        0.23700
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00980, CPU: 0.00882]


item_cat_seqs_bra_0_10748.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:19, written at 23:37:21
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          0.39        0.37           1        0.39300        0.37000
  Solutions by Category                      0.01        0.00           1        0.00600        0.00000
  Pop_Table_Iterate_Base                     0.39        0.38           1        0.38900        0.38000
  Pop_Table_Iterate_Link                     0.41        0.39           1        0.40500        0.39000
  Pop_Table_Iterate_Link - item level        0.02        0.00           1        0.01500        0.00000
  Pop_Table_Iterate_Base_Link                0.39        0.36           1        0.39300        0.36000
  Array_Iterate                              0.40        0.39           1        0.39900        0.39000
  Pop_Table_Recurse                          0.38        0.36           1        0.38200        0.36000
  Array_Recurse                              0.42        0.40           1        0.41500        0.40000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      2.80        2.65          10        0.27980        0.26500
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00917, CPU: 0.00826]


item_cat_seqs_rsf_bra_10_0.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:37:22, written at 23:37:24
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                         0.69        0.65           1        0.69200        0.65000
  rsf_sql_material_v                0.30        0.26           1        0.29700        0.26000
  Item_Cat_Seqs.Init                0.02        0.01           3        0.00533        0.00333
  rsf_irk_irs_tabs_v                0.11        0.08           1        0.10900        0.08000
  rsf_irk_tab_where_fun_ts_v        0.46        0.41           1        0.46300        0.41000
  rsf_irk_tab_where_fun_v           0.15        0.12           1        0.14700        0.12000
  (Other)                           1.14        1.07           1        1.13500        1.07000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                             2.86        2.60           9        0.31767        0.28889
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00901, CPU: 0.00901]


item_cat_seqs_rsf_bra_100_0.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:37:25, written at 23:37:42
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                         6.53        6.31           1        6.52700        6.31000
  rsf_sql_material_v                2.85        2.75           1        2.84500        2.75000
  Item_Cat_Seqs.Init                0.02        0.00           3        0.00533        0.00000
  rsf_irk_irs_tabs_v                1.14        1.10           1        1.14300        1.10000
  rsf_irk_tab_where_fun_ts_v        3.79        3.77           1        3.79200        3.77000
  rsf_irk_tab_where_fun_v           1.60        1.57           1        1.60200        1.57000
  (Other)                           1.14        1.11           1        1.14100        1.11000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                            17.07       16.61           9        1.89622        1.84556
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00885, CPU: 0.00796]


item_cat_seqs_rsf_bra_100_10748.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:37:42, written at 23:37:44
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                         0.35        0.32           1        0.34700        0.32000
  rsf_sql_material_v                0.16        0.15           1        0.15900        0.15000
  Item_Cat_Seqs.Init                0.01        0.01           3        0.00433        0.00333
  rsf_irk_irs_tabs_v                0.05        0.02           1        0.05000        0.02000
  rsf_irk_tab_where_fun_ts_v        0.78        0.73           1        0.77600        0.73000
  rsf_irk_tab_where_fun_v           0.17        0.14           1        0.17300        0.14000
  (Other)                           1.14        1.13           1        1.14100        1.13000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                             2.66        2.50           9        0.29544        0.27778
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00877, CPU: 0.00789]


item_cat_seqs_rsf_bra_0_10748.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:37:45, written at 23:37:49
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                         0.71        0.68           1        0.70700        0.68000
  rsf_sql_material_v                0.41        0.38           1        0.41400        0.38000
  Item_Cat_Seqs.Init                0.01        0.01           3        0.00467        0.00333
  rsf_irk_irs_tabs_v                0.10        0.10           1        0.10300        0.10000
  rsf_irk_tab_where_fun_ts_v        1.99        1.97           1        1.98700        1.97000
  rsf_irk_tab_where_fun_v           0.46        0.43           1        0.45800        0.43000
  (Other)                           1.16        1.10           1        1.15600        1.10000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                             4.84        4.67           9        0.53767        0.51889
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00971, CPU: 0.00971]


item_cat_seqs_loop_bra_10_3.log

> Timer Set: Item_Cat_Seqs_Loop, Constructed at 10 Jun 2024 23:37:49, written at 23:37:52
  =======================================================================================
  Timer                                       Elapsed         CPU       Calls       Ela/Call       CPU/Call
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Iteratively_Refine_Recurse - path level        0.82        0.80           1        0.81900        0.80000
  Iteratively_Refine_Recurse - item level        0.81        0.78           1        0.81000        0.78000
  Iteratively_Refine_Iterate - path level        0.78        0.77           1        0.78000        0.77000
  Iteratively_Refine_RSF - path level            0.46        0.42           1        0.45800        0.42000
  (Other)                                        0.00        0.00           1        0.00100        0.00000
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                          2.87        2.77           5        0.57360        0.55400
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00901, CPU: 0.00811]


item_cat_seqs_eng_50_0.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:37:54, written at 23:38:04
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          1.46        1.42           1        1.45500        1.42000
  Solutions by Category                      0.02        0.00           1        0.01900        0.00000
  Pop_Table_Iterate_Base                     1.36        1.30           1        1.35600        1.30000
  Pop_Table_Iterate_Link                     1.36        1.32           1        1.35600        1.32000
  Pop_Table_Iterate_Link - item level        0.02        0.01           1        0.02200        0.01000
  Pop_Table_Iterate_Base_Link                1.23        1.15           1        1.23100        1.15000
  Array_Iterate                              1.76        1.72           1        1.76200        1.72000
  Pop_Table_Recurse                          1.40        1.39           1        1.40100        1.39000
  Array_Recurse                              1.73        1.69           1        1.72700        1.69000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                     10.33       10.00          10        1.03300        1.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00935, CPU: 0.00841]


item_cat_seqs_eng_300_0.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:38:04, written at 23:38:55
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          7.14        6.50           1        7.13600        6.50000
  Solutions by Category                      0.01        0.00           1        0.00600        0.00000
  Pop_Table_Iterate_Base                     6.55        6.14           1        6.54600        6.14000
  Pop_Table_Iterate_Link                     6.26        5.97           1        6.26300        5.97000
  Pop_Table_Iterate_Link - item level        0.02        0.00           1        0.02100        0.00000
  Pop_Table_Iterate_Base_Link                5.16        5.13           1        5.16300        5.13000
  Array_Iterate                              9.13        8.08           1        9.13200        8.08000
  Pop_Table_Recurse                          7.15        6.37           1        7.14500        6.37000
  Array_Recurse                              8.98        8.14           1        8.98100        8.14000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                     50.39       46.33          10        5.03940        4.63300
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00935, CPU: 0.00841]


item_cat_seqs_eng_300_1952.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:38:55, written at 23:39:03
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                          1.08        1.04           1        1.07600        1.04000
  Solutions by Category                      0.01        0.00           1        0.00600        0.00000
  Pop_Table_Iterate_Base                     1.07        1.02           1        1.06500        1.02000
  Pop_Table_Iterate_Link                     1.06        1.01           1        1.05600        1.01000
  Pop_Table_Iterate_Link - item level        0.02        0.00           1        0.01500        0.00000
  Pop_Table_Iterate_Base_Link                1.01        0.96           1        1.00800        0.96000
  Array_Iterate                              1.35        1.31           1        1.34500        1.31000
  Pop_Table_Recurse                          1.09        1.05           1        1.09000        1.05000
  Array_Recurse                              1.42        1.41           1        1.42200        1.41000
  (Other)                                    0.00        0.00           1        0.00100        0.00000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                      8.08        7.80          10        0.80840        0.78000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00943, CPU: 0.00943]


item_cat_seqs_eng_0_1952.log

> Timer Set: Item_Cat_Seqs, Constructed at 10 Jun 2024 23:39:03, written at 23:50:22
  ==================================================================================
  Timer                                   Elapsed         CPU       Calls       Ela/Call       CPU/Call
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Pop_Table_Iterate                         80.87       78.89           1       80.87200       78.89000
  Solutions by Category                      0.05        0.03           1        0.04700        0.03000
  Pop_Table_Iterate_Base                    87.28       82.65           1       87.27900       82.65000
  Pop_Table_Iterate_Link                    86.37       84.77           1       86.37100       84.77000
  Pop_Table_Iterate_Link - item level        1.40        1.38           1        1.40300        1.38000
  Pop_Table_Iterate_Base_Link               84.90       83.68           1       84.90300       83.68000
  Array_Iterate                            115.81      112.15           1      115.80500      112.15000
  Pop_Table_Recurse                         82.56       81.07           1       82.55600       81.07000
  Array_Recurse                            139.69      138.17           1      139.69200      138.17000
  (Other)                                    0.00        0.01           1        0.00200        0.01000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                    678.93      662.80          10       67.89300       66.28000
  -----------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.01099, CPU: 0.00989]


item_cat_seqs_rsf_eng_50_0.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:50:22, written at 23:55:22
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                       207.53      205.83           1      207.52500      205.83000
  rsf_sql_material_v               51.81       50.81           1       51.80900       50.81000
  Item_Cat_Seqs.Init                0.04        0.03           3        0.01300        0.01000
  rsf_irk_irs_tabs_v                8.91        8.75           1        8.91200        8.75000
  rsf_irk_tab_where_fun_ts_v       19.57       19.38           1       19.56500       19.38000
  rsf_irk_tab_where_fun_v          10.70       10.62           1       10.69800       10.62000
  (Other)                           1.17        1.14           1        1.16600        1.14000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                           299.71      296.56           9       33.30156       32.95111
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00885, CPU: 0.00796]


item_cat_seqs_rsf_eng_300_0.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 10 Jun 2024 23:55:22, written at 00:26:21
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                      1252.52     1234.32           1     1252.52200     1234.32000
  rsf_sql_material_v              318.43      305.14           1      318.42600      305.14000
  Item_Cat_Seqs.Init                0.03        0.00           3        0.01000        0.00000
  rsf_irk_irs_tabs_v               72.18       63.97           1       72.18400       63.97000
  rsf_irk_tab_where_fun_ts_v      132.68      125.39           1      132.67700      125.39000
  rsf_irk_tab_where_fun_v          82.30       75.52           1       82.29500       75.52000
  (Other)                           1.19        1.11           1        1.19400        1.11000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                          1859.33     1805.45           9      206.59200      200.60556
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00885, CPU: 0.00973]


item_cat_seqs_rsf_eng_300_1952.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 11 Jun 2024 00:26:21, written at 00:28:47
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                        79.78       78.63           1       79.78100       78.63000
  rsf_sql_material_v               20.43       19.73           1       20.42900       19.73000
  Item_Cat_Seqs.Init                0.04        0.01           3        0.01200        0.00333
  rsf_irk_irs_tabs_v                3.10        3.04           1        3.09500        3.04000
  rsf_irk_tab_where_fun_ts_v       32.27       32.04           1       32.26900       32.04000
  rsf_irk_tab_where_fun_v           8.31        8.27           1        8.31300        8.27000
  (Other)                           1.20        1.12           1        1.20100        1.12000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                           145.12      142.84           9       16.12489       15.87111
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.01064, CPU: 0.01064]


item_cat_seqs_rsf_eng_0_1952.log

> Timer Set: Item_Cat_Seqs_RSF, Constructed at 11 Jun 2024 00:28:47, written at 03:35:34
  ======================================================================================
  Timer                          Elapsed         CPU       Calls       Ela/Call       CPU/Call
  --------------------------  ----------  ----------  ----------  -------------  -------------
  rsf_sql_v                      2288.45     1375.34           1     2288.45100     1375.34000
  rsf_sql_material_v             1727.05      605.12           1     1727.04900      605.12000
  Item_Cat_Seqs.Init                0.03        0.01           3        0.01100        0.00333
  rsf_irk_irs_tabs_v              214.03      208.70           1      214.03200      208.70000
  rsf_irk_tab_where_fun_ts_v     5795.63     5745.64           1     5795.63100     5745.64000
  rsf_irk_tab_where_fun_v        1181.32     1169.62           1     1181.31800     1169.62000
  (Other)                           1.19        1.11           1        1.19300        1.11000
  --------------------------  ----------  ----------  ----------  -------------  -------------
  Total                         11207.71     9105.54           9     1245.30078     1011.72667
  --------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.00990, CPU: 0.01089]


item_cat_seqs_loop_eng_50_3.log

> Timer Set: Item_Cat_Seqs_Loop, Constructed at 11 Jun 2024 03:35:34, written at 03:35:53
  =======================================================================================
  Timer                                       Elapsed         CPU       Calls       Ela/Call       CPU/Call
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Iteratively_Refine_Recurse - path level        2.77        2.73           1        2.76600        2.73000
  Iteratively_Refine_Recurse - item level        2.77        2.74           1        2.76800        2.74000
  Iteratively_Refine_Iterate - path level        2.28        2.18           1        2.27500        2.18000
  Iteratively_Refine_RSF - path level           10.53       10.42           1       10.52800       10.42000
  (Other)                                        0.00        0.00           1        0.00100        0.00000
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  Total                                         18.34       18.07           5        3.66760        3.61400
  ---------------------------------------  ----------  ----------  ----------  -------------  -------------
  [Timer timed (per call in ms): Elapsed: 0.01053, CPU: 0.00947]


Total time taken: 14334 seconds
</pre>

</div>

#### Run Instance 3 Results Folder - results_03
[&uarr; Output Logs](#output-logs)<br />

<img src="/images/2024/08/18/results_03.png">

## 3 Code Instrumentation
[&uarr; Contents](#contents)<br />
[&darr; Code Timing](#code-timing)<br />
[&darr; Execution Plans](#execution-plans)<br />

Code instrumentation allows for the automatic gathering of information about a program's execution, including:

- Elapsed and CPU timings, and numbers of iterations, of code sections
- Any information of interest available at runtime
- Execution plans for SQL statements

Two GitHub projects provide useful packages to help with this:
- [Utils - Oracle PL/SQL General Utilities Module](https://github.com/BrenPatF/oracle_plsql_utils)
- [Oracle PL/SQL Code Timing Module](https://github.com/BrenPatF/timer_set_oracle)

I have also created code timing modules on GitHub in Javascript, Python and Powershell.

### Code Timing
[&uarr; 3 Code Instrumentation](#3-code-instrumentation)<br />
[&darr; Code Timing Example Output](#code-timing-example-output)<br />
[&darr; Code Timing Example Code](#code-timing-example-code)<br />

#### Code Timing Example Output
[&uarr; Code Timing](#code-timing)<br />

Here is an extract from the sixth article, [6. Mixed SQL and PL/SQL Methods for Item/Category Optimization](../6_mixed_sql_plsql_for_itemcategory_optimization/6_mixed_sql_plsql_for_itemcategory_optimization.md#code-timing):

<blockquote>Each solution method was instrumented using the author's own code timing package [Oracle PL/SQL Code Timing Module]. Here are the results for the Pop_Table_Iterate solution. The INSERT and DELETE statements are timed separately at each iteration. The code timing package allows the timer name to include the rows processed, as shown below, providing quite detailed statistics at each iteration within PL/SQL.</blockquote>

Here are the results reported there for Pop_Table_Iterate on the England dataset:
```
Timer Set: Pop_Table_Iterate, Constructed at 10 Jun 2024 23:39:03, written at 23:40:24
======================================================================================
Timer                        Elapsed         CPU       Calls       Ela/Call       CPU/Call
------------------------  ----------  ----------  ----------  -------------  -------------
Initial delete (0)              0.01        0.01           1        0.00900        0.01000
Initial insert (1)              0.00        0.00           1        0.00000        0.00000
Insert paths 1 (17)             0.00        0.00           1        0.00000        0.00000
Delete paths 1 (1)              0.00        0.00           1        0.00000        0.00000
Insert paths 2 (290)            0.00        0.01           1        0.00400        0.01000
Delete paths 2 (17)             0.00        0.00           1        0.00000        0.00000
Insert paths 3 (553)            0.03        0.02           1        0.03100        0.02000
Delete paths 3 (290)            0.00        0.00           1        0.00000        0.00000
Insert paths 4 (279)            0.06        0.06           1        0.05600        0.06000
Delete paths 4 (553)            0.00        0.00           1        0.00000        0.00000
Insert paths 5 (1252)           0.03        0.03           1        0.03100        0.03000
Delete paths 5 (279)            0.00        0.00           1        0.00000        0.00000
Insert paths 6 (15349)          0.12        0.11           1        0.12000        0.11000
Delete paths 6 (1252)           0.00        0.01           1        0.00200        0.01000
Insert paths 7 (110100)         1.19        1.12           1        1.18700        1.12000
Delete paths 7 (15349)          0.02        0.01           1        0.01500        0.01000
Insert paths 8 (460638)         6.32        6.11           1        6.31600        6.11000
Delete paths 8 (110100)         0.11        0.07           1        0.10700        0.07000
Insert paths 9 (864748)        21.31       20.49           1       21.31400       20.49000
Delete paths 9 (460638)         0.41        0.30           1        0.41000        0.30000
Insert paths 10 (389615)       35.84       35.50           1       35.83500       35.50000
Delete paths 10 (864748)        0.68        0.42           1        0.67900        0.42000
Insert paths 11 (50)           14.29       14.24           1       14.29100       14.24000
Delete paths 11 (389615)        0.31        0.22           1        0.30500        0.22000
(Other)                         0.00        0.00           1        0.00000        0.00000
------------------------  ----------  ----------  ----------  -------------  -------------
Total                          80.71       78.73          25        3.22848        3.14920
------------------------  ----------  ----------  ----------  -------------  -------------
[Timer timed (per call in ms): Elapsed: 0.00952, CPU: 0.01143]
```

#### Code Timing Example Code
[&uarr; Code Timing](#code-timing)<br />
[&darr; Pop_Table_Iterate Extract](#pop_table_iterate-extract)<br />
[&darr; insert_Initial_Path](#insert_initial_path)<br />
[&darr; insert_Paths Extract](#insert_paths-extract)<br />

##### Pop_Table_Iterate Extract
[&uarr; Code Timing Example Code](#code-timing-example-code)<br />

Pop_Table_Iterate is one of the entry point procedures, and the extracts below show how the timer set is constructed, passed to the two called procedures, insert_Initial_Path and insert_Paths, and the results as shown above are written to log.

```sql
  l_timer_set       PLS_INTEGER := Timer_Set.Construct ('Pop_Table_Iterate');
BEGIN
...
  insert_Initial_Path(p_timer_set => l_timer_set);
  FOR i IN 1..g_seq_size LOOP
    insert_Paths(p_timer_set => l_timer_set,
                 p_iter      => i);
  END LOOP;
  Utils.W(Timer_Set.Format_Results(l_timer_set));
...
```

##### insert_Initial_Path
[&uarr; Code Timing Example Code](#code-timing-example-code)<br />

This is the complete code for the first procedure called, showing the two calls to Timer_Set.Increment_Time that give rise to the output lines above:

```sql
PROCEDURE insert_Initial_Path(
            p_timer_set                    PLS_INTEGER) IS
BEGIN
  DELETE paths;
  Timer_Set.Increment_Time (p_timer_set, 'Initial delete (' || SQL%ROWCOUNT || ')');
  INSERT INTO paths (
     path_rnk, item_rnk, lev, tot_price, tot_value, cat_id, next_cat_id, same_cats, min_items, cats_path, path
  )
  SELECT 0, 0, 0, 0, 0, 'AL', cat_id, 0, 0, '',''
    FROM items_ranked
   WHERE item_rnk = 1;
  Timer_Set.Increment_Time (p_timer_set, 'Initial insert (' || SQL%ROWCOUNT || ')');
END insert_Initial_Path;
```

The output is copied below:
```
Initial delete (0)              0.01        0.01           1        0.00900        0.01000
Initial insert (1)              0.00        0.00           1        0.00000        0.00000
```

##### insert_Paths Extract
[&uarr; Code Timing Example Code](#code-timing-example-code)<br />

The extracts below show the two calls to Timer_Set.Increment_Time that give rise to the output lines above, for each iteration.

```sql
...
  l_n_rows := SQL%ROWCOUNT;
...
  Timer_Set.Increment_Time (p_timer_set, 'Insert paths ' || p_iter || ' (' || l_n_rows || ')');
  DELETE paths WHERE lev = p_iter - 1;
  Timer_Set.Increment_Time (p_timer_set, 'Delete paths ' || p_iter || ' (' || SQL%ROWCOUNT || ')');
```

The output for the first iteration is copied below:
```
Insert paths 1 (17)             0.00        0.00           1        0.00000        0.00000
Delete paths 1 (1)              0.00        0.00           1        0.00000        0.00000
```
### Execution Plans
[&uarr; 3 Code Instrumentation](#3-code-instrumentation)<br />
[&darr; Statement with Hint](#statement-with-hint)<br />
[&darr; insert_Paths Extract](#insert_paths-extract-1)<br />
[&darr; Execution Plan - Pop_Table_Iterate: England dataset, 10'th iteration](#execution-plan---pop_table_iterate-england-dataset-10th-iteration)<br />

Oracle provides mechanisms for obtaining the execution plan it used for an SQL statement. A common way to do this is to use a hint, gather_plan_statistics, in the statement, that provides for the gathering of row count and other statistics in the execution, and then to call an API from the sys package DBMS_XPlan to display the plan.

People often do this manually, but I prefer to do it programmatically, using a wrapper utility from my GitHub project, [Utils - Oracle PL/SQL General Utilities Module](https://github.com/BrenPatF/oracle_plsql_utils). This has several advantages:
- The plan can age out of the system storage area, but that's unlikely if the API call immediately succeeds the statement execution
- When the statement occurs repeatedly within a PL/SQL program, logic can be included to display the plan only for a specific execution; in our case, code timing told us that for the England dataset the 10'th iteration took longest, so it makes sense to look at the plan for that iteration
- It's automated, so requires no effort to repeat

#### Statement with Hint
[&uarr; Execution Plans](#execution-plans)<br />

```sql
  INSERT /*+ gather_plan_statistics INSERT_PATHS */ INTO paths (
    path_rnk, item_rnk, lev, tot_price, tot_value, cat_id, next_cat_id, same_cats, min_items, cats_path, path
  )
  WITH path_join AS (
  ...
```

#### insert_Paths Extract
[&uarr; Execution Plans](#execution-plans)<br />

The plan is displayed only for the 10'th iteration, and the wrapper utility looks for the plan with a marker string mentioned in the statement hint.

```sql
  IF p_iter = 10 THEN
    Utils.W(Utils.Get_XPlan(p_sql_marker => 'INSERT_PATHS'));
  END IF;
```

#### Execution Plan - Pop_Table_Iterate: England dataset, 10'th iteration
[&uarr; Execution Plans](#execution-plans)<br />

```
Plan hash value: 2684864324
----------------------------------------------------------------------------------------------------------------------------------------------
| Id  | Operation                | Name               | Starts | E-Rows | A-Rows |   A-Time   | Buffers |  OMem |  1Mem | Used-Mem | Used-Tmp|
----------------------------------------------------------------------------------------------------------------------------------------------
|   0 | INSERT STATEMENT         |                    |      1 |        |      0 |00:00:36.16 |     922K|       |       |          |         |
|   1 |  LOAD TABLE CONVENTIONAL | PATHS              |      1 |        |      0 |00:00:36.16 |     922K|       |       |          |         |
|*  2 |   VIEW                   |                    |      1 |      1 |    389K|00:00:35.94 |     890K|       |       |          |         |
|   3 |    WINDOW SORT           |                    |      1 |      1 |    389K|00:00:35.76 |     890K|    82M|  3117K|   73M (0)|         |
|   4 |     NESTED LOOPS         |                    |      1 |      1 |    389K|00:00:12.54 |     890K|       |       |          |         |
|*  5 |      TABLE ACCESS FULL   | PATHS              |      1 |      1 |    864K|00:00:00.15 |   16167 |       |       |          |         |
|*  6 |      INDEX RANGE SCAN    | SYS_IOT_TOP_177213 |    864K|      1 |    389K|00:00:32.73 |     874K|       |       |          |         |
----------------------------------------------------------------------------------------------------------------------------------------------
Predicate Information (identified by operation id):
---------------------------------------------------
2 - filter(("PATH_RNK"<=:B7 OR :B7=0))
5 - filter("TRW"."LEV"<:B1)
6 - access("IRK"."ITEM_RNK">="TRW"."ITEM_RNK"+1)
filter(("TRW"."TOT_PRICE"+"IRK"."ITEM_PRICE"+:B5<=:B4 AND "TRW"."TOT_VALUE"+"IRK"."ITEM_VALUE"+:B3>=:B2 AND
"IRK"."ITEM_RNK"<="IRK"."N_ITEMS"-(:B1-"TRW"."LEV"-1) AND "IRK"."MAX_ITEMS">=CASE "IRK"."CAT_ID" WHEN "TRW"."CAT_ID" THEN
"TRW"."SAME_CATS"+1 ELSE 1 END  AND "IRK"."MIN_REMAIN"<=:B1-("TRW"."LEV"+1)+LEAST(CASE "IRK"."CAT_ID" WHEN "TRW"."CAT_ID" THEN
"TRW"."SAME_CATS"+1 ELSE 1 END ,"IRK"."MIN_ITEMS") AND ("IRK"."CAT_ID"="TRW"."CAT_ID" OR "TRW"."SAME_CATS">="TRW"."MIN_ITEMS") AND
("IRK"."CAT_ID"="TRW"."CAT_ID" OR "IRK"."CAT_ID"=NVL("TRW"."NEXT_CAT_ID","IRK"."CAT_ID"))))
Note
-----
- dynamic statistics used: dynamic sampling (level=2)
```

## 4 Unit Testing
[&uarr; Contents](#contents)<br />
[&darr; Input Side](#input-side)<br />
[&darr; Output Side](#output-side)<br />

Unit testing is a key part of the verification process, and is fully automated. Automation means that regression testing can be performed in seconds when any change is made, and also tends to improve test quality in general.

The seventh article has a detailed description of our unit testing process, [OPICO 7: Verification/Unit Testing](https://brenpatf.github.io/2024/08/11/opico-7_verification.html#4-unit-testing), and we'll make a high level summary here.

Our approach, following [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html), has particular advantages. Here are its key features:
- Test scenarios are specified in a JSON file, so that no programming is required to add scenarios once the necessary wrapper function is written
- The JSON file includes both inputs and expected outputs and the testing framework reports success or failure based on these
- All solution methods share a single JSON scenarios file
- Each solution method has its own wrapper API that is called by the testing framework, but the wrappers share a base function that does the setup of test data and makes the necessary calls, and so are very simple
- Running the unit test driver script produces:
    - a summary listing of results for all units tested
    - a subfolder for each unit, containing a summary HTML report for that unit with links to...
    - scenario HTML pages listing all inputs, expected outputs and any actual outputs that differ

### Input Side
[&uarr; 4 Unit Testing](#4-unit-testing)<br />
[&darr; Example Wrapper API Function - Array_Iterate](#example-wrapper-api-function---array_iterate)<br />
[&darr; Shared JSON File](#shared-json-file)<br />

Here we list some of the main inputs to the unit testing process.

#### Example Wrapper API Function - Array_Iterate
[&uarr; Input Side](#input-side)<br />

This is the Array_Iterate wrapper API function called by the library unit test package to return actual outputs for a given scenario. It calls a common function, purely_Wrap_Best_Combis, that centralises the logic.

```sql
FUNCTION Array_Iterate(
              p_inp_3lis                     L3_chr_arr)
              RETURN                         L2_chr_arr IS
BEGIN
    RETURN purely_Wrap_Best_Combis(
              p_view_name   => 'array_iterate_v',
              p_inp_3lis    => p_inp_3lis,
              p_proc_name   => 'Init');
END Array_Iterate;
```

#### Shared JSON File
[&uarr; Input Side](#input-side)<br />

This is the JSON file that is used by the unit testing process for all units, containing all test scenarios and metadata. The title is superseded by the specific title for each unit test, stored in a database table.

<div class="scrollbox">

<pre>
{
  "meta": {
    "title": "Best Combis - Pre-Emptive - Item Categories",
    "delimiter": "|",
    "inp": {
      "Category": [
        "Id",
        "Min#",
        "Max#"
      ],
      "Item": [
        "Id",
        "Category",
        "Price",
        "Value"
      ],
      "Scalars": [
        "Size",
        "Max Price",
        "Top N",
        "Min Value",
        "Keep#"
      ]
    },
    "out": {
      "Solution": [
        "Path",
        "Total Price",
        "Total Value"
      ]
    }
  },
  "scenarios": {
    "Choose 1 / n": {
      "active_yn": "Y",
      "category_set": "Choose Range (r / n)",
      "inp": {
        "Category": [
          "C1|0|2",
          "AL|1|1"
        ],
        "Item": [
          "I1|C1|5|5",
          "I2|C1|6|6"
        ],
        "Scalars": [
          "1|20|1|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I2|6|6"
        ]
      }
    },
    "Choose r / n (r < n)": {
      "active_yn": "Y",
      "category_set": "Choose Range (r / n)",
      "inp": {
        "Category": [
          "C1|0|2",
          "AL|2|2"
        ],
        "Item": [
          "I1|C1|5|15",
          "I2|C1|6|16",
          "I3|C1|7|17"
        ],
        "Scalars": [
          "2|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I2I3|13|33",
          "I1I3|12|32",
          "I1I2|11|31"
        ]
      }
    },
    "Choose n / n": {
      "active_yn": "Y",
      "category_set": "Choose Range (r / n)",
      "inp": {
        "Category": [
          "C1|0|3",
          "AL|3|3"
        ],
        "Item": [
          "I1|C1|5|15",
          "I2|C1|6|16",
          "I3|C1|7|17"
        ],
        "Scalars": [
          "3|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I1I2I3|18|48"
        ]
      }
    },
    "No itemcats": {
      "active_yn": "Y",
      "category_set": "ItemCat Multiplicity",
      "inp": {
        "Category": [
          "C1|0|1000",
          "AL|3|3"
        ],
        "Item": [
          "I1|C1|5|15",
          "I2|C1|6|16",
          "I3|C1|7|17"
        ],
        "Scalars": [
          "3|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I1I2I3|18|48"
        ]
      }
    },
    "One itemcat": {
      "active_yn": "Y",
      "category_set": "ItemCat Multiplicity",
      "inp": {
        "Category": [
          "C1|0|2",
          "AL|2|2"
        ],
        "Item": [
          "I1|C1|5|15",
          "I2|C1|6|16",
          "I3|C1|7|17"
        ],
        "Scalars": [
          "2|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I2I3|13|33",
          "I1I3|12|32",
          "I1I2|11|31"
        ]
      }
    },
    "Multiple itemcats": {
      "active_yn": "Y",
      "category_set": "ItemCat Multiplicity",
      "inp": {
        "Category": [
          "C1|0|2",
          "C2|0|2",
          "AL|2|2"
        ],
        "Item": [
          "I1|C1|5|15",
          "I2|C1|6|16",
          "I3|C2|7|17"
        ],
        "Scalars": [
          "2|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I2I3|13|33",
          "I1I3|12|32",
          "I1I2|11|31"
        ]
      }
    },
    "Itemcat min only": {
      "active_yn": "Y",
      "category_set": "ItemCat Range",
      "inp": {
        "Category": [
          "C1|1|2",
          "C2|0|2",
          "AL|1|1"
        ],
        "Item": [
          "I1|C1|5|5",
          "I2|C2|10|10"
        ],
        "Scalars": [
          "1|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I1|5|5"
        ]
      }
    },
    "Itemcat max only": {
      "active_yn": "Y",
      "category_set": "ItemCat Range",
      "inp": {
        "Category": [
          "C1|0|2",
          "C2|0|0",
          "AL|1|1"
        ],
        "Item": [
          "I1|C1|5|5",
          "I2|C2|10|10"
        ],
        "Scalars": [
          "1|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
          "I1|5|5"
        ]
      }
    },
    "Itemcat with min < max": {
      "active_yn": "Y",
      "category_set": "ItemCat Range",
      "inp": {
        "Category": [
           "C1|1|2",
           "C2|1|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|0|10"
        ],
        "Scalars": [
           "2|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|10|20",
           "I1I2|15|15"
        ]
      }
    },
    "Itemcat with min = max": {
      "active_yn": "Y",
      "category_set": "ItemCat Range",
      "inp": {
        "Category": [
           "C1|1|1",
           "C2|0|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|10"
        ],
        "Scalars": [
           "2|20|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|20",
           "I1I2|15|15"
        ]
      }
    },
    "No solution (want top N)": {
      "active_yn": "Y",
      "category_set": "Solution Multiplicity (Actual / Top N)",
      "inp": {
        "Category": [
           "C1|1|2",
           "C2|1|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|10"
        ],
        "Scalars": [
           "2|10|3|0|10"
        ]
      },
      "out": {
        "Solution": [
        ]
      }
    },
    "1 solution (want top 1)": {
      "active_yn": "Y",
      "category_set": "Solution Multiplicity (Actual / Top N)",
      "inp": {
        "Category": [
           "C1|1|2",
           "C2|0|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|10"
        ],
        "Scalars": [
           "2|12|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I1I3|12|15"
        ]
      }
    },
    "N solutions (want top N)": {
      "active_yn": "Y",
      "category_set": "Solution Multiplicity (Actual / Top N)",
      "inp": {
        "Category": [
           "C1|0|2",
           "C2|0|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|25|2|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|25",
           "I1I3|12|20"
        ]
      }
    },
    "< N solutions (want top N)": {
      "active_yn": "Y",
      "category_set": "Solution Multiplicity (Actual / Top N)",
      "inp": {
        "Category": [
           "C1|0|2",
           "C2|0|2",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|15|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I1I3|12|20",
           "I1I2|15|15"
        ]
      }
    },
    "No active constraint": {
      "active_yn": "Y",
      "category_set": "Constraint Activity",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|0|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|25",
           "I1I3|12|20",
           "I1I2|15|15"
        ]
      }
    },
    "Price maximum active": {
      "active_yn": "Y",
      "category_set": "Constraint Activity",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|0|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|15|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I1I3|12|20",
           "I1I2|15|15"
        ]
      }
    },
    "Itemcat minimum active": {
      "active_yn": "Y",
      "category_set": "Constraint Activity",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|1|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|25",
           "I1I2|15|15"
        ]
      }
    },
    "Itemcat maximum active": {
      "active_yn": "Y",
      "category_set": "Constraint Activity",
      "inp": {
        "Category": [
           "C1|0|1",
           "C2|0|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|25",
           "I1I2|15|15"
        ]
      }
    },
    "Solutions exist": {
      "active_yn": "Y",
      "category_set": "Constraint Infeasibility",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|1|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
           "I2I3|17|25",
           "I1I2|15|15"
        ]
      }
    },
    "No solution - price maximum": {
      "active_yn": "Y",
      "category_set": "Constraint Infeasibility",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|1|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|11|3|0|10"
        ]
      },
      "out": {
        "Solution": [
        ]
      }
    },
    "No solution - itemcat minimum": {
      "active_yn": "Y",
      "category_set": "Constraint Infeasibility",
      "inp": {
        "Category": [
           "C1|0|3",
           "C2|2|3",
           "AL|2|2"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "2|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
        ]
      }
    },
    "No solution - itemcat maximum": {
      "active_yn": "Y",
      "category_set": "Constraint Infeasibility",
      "inp": {
        "Category": [
           "C1|0|1",
           "C2|1|1",
           "AL|3|3"
        ],
        "Item": [
           "I1|C1|5|5",
           "I2|C2|10|10",
           "I3|C1|7|15"
        ],
        "Scalars": [
           "3|50|3|0|10"
        ]
      },
      "out": {
        "Solution": [
        ]
      }
    }
  }
}

</pre>
</div>

### Output Side
[&uarr; 4 Unit Testing](#4-unit-testing)<br />
[&darr; Results Summary for All Solution Methods](#results-summary-for-all-solution-methods)<br />
[&darr; Unit Test Unit Folders](#unit-test-unit-folders)<br />
[&darr; Unit Test Report: Best Item Category Combis - Array_Iterate](#unit-test-report-best-item-category-combis---array_iterate)<br />
[&darr; Scenario 2: Choose r / n (r < n) [Category Set: Choose Range (r / n)]](#scenario-2-choose-r--n-r--n-category-set-choose-range-r--n)<br />

Here we show the main outputs from the unit testing process.

#### Results Summary for All Solution Methods
[&uarr; Output Side](#output-side)<br />

The script creates a results subfolder for each unit in the 'item_cat_seqs' group, with results in text and HTML formats, in the script folder, and outputs the following summary for the 15 methods tested:

<div class="scrollbox">

<pre>
File:          tt_item_cat_seqs.rsf_post_valid_out.json
Title:         Best Item Category Combis - RSF_Post_Valid
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---rsf_post_valid

File:          tt_item_cat_seqs.rsf_sql_out.json
Title:         Best Item Category Combis - RSF_SQL
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---rsf_sql

File:          tt_item_cat_seqs.rsf_sql_material_out.json
Title:         Best Item Category Combis - RSF_SQL_Material
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---rsf_sql_material

File:          tt_item_cat_seqs.rsf_irk_irs_tabs_out.json
Title:         Best Item Category Combis - RSF_Irk_IRS_Tabs
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---rsf_irk_irs_tabs

File:          tt_item_cat_seqs.rsf_irk_tab_where_fun_out.json
Title:         Best Item Category Combis - RSF_Irk_Tab_Where_Fun
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---rsf_irk_tab_where_fun

File:          tt_item_cat_seqs.pop_table_iterate_out.json
Title:         Best Item Category Combis - Pop_Table_Iterate
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---pop_table_iterate

File:          tt_item_cat_seqs.pop_table_iterate_base_out.json
Title:         Best Item Category Combis - Pop_Table_Iterate_Base
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---pop_table_iterate_base

File:          tt_item_cat_seqs.pop_table_iterate_link_out.json
Title:         Best Item Category Combis - Pop_Table_Iterate_Link
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---pop_table_iterate_link

File:          tt_item_cat_seqs.pop_table_iterate_base_link_out.json
Title:         Best Item Category Combis - Pop_Table_Iterate_Base_Link
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---pop_table_iterate_base_link

File:          tt_item_cat_seqs.array_iterate_out.json
Title:         Best Item Category Combis - Array_Iterate
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---array_iterate

File:          tt_item_cat_seqs.pop_table_recurse_out.json
Title:         Best Item Category Combis - Pop_Table_Recurse
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---pop_table_recurse

File:          tt_item_cat_seqs.array_recurse_out.json
Title:         Best Item Category Combis - Array_Recurse
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---array_recurse

File:          tt_item_cat_seqs.iteratively_refine_recurse_out.json
Title:         Best Item Category Combis - Iteratively_Refine_Recurse
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---iteratively_refine_recurse

File:          tt_item_cat_seqs.iteratively_refine_iterate_out.json
Title:         Best Item Category Combis - Iteratively_Refine_Iterate
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---iteratively_refine_iterate

File:          tt_item_cat_seqs.iteratively_refine_rsf_out.json
Title:         Best Item Category Combis - Iteratively_Refine_RSF
Inp Groups:    3
Out Groups:    2
Tests:         22
Fails:         0
Folder:        best-item-category-combis---iteratively_refine_rsf
</pre>
</div>

#### Unit Test Unit Folders
[&uarr; Output Side](#output-side)<br />

The screenshot shows the 15 unit test folders created by running the unit test driver script, with the right hand pane showing the results files for the Array_Iterate method: a single text format file, and HTML files for the results summary and the 22 individual scenario pages.

<img src="/images/2024/08/18/UT-Results-Folders.png">

#### Unit Test Report: Best Item Category Combis - Array_Iterate
[&uarr; Output Side](#output-side)<br />

Here is the results summary for one of the solution methods, in HTML format:
<img src="/images/2024/08/18/UT-Summary.png">

#### Scenario 2: Choose r / n (r < n) [Category Set: Choose Range (r / n)]
[&uarr; Output Side](#output-side)<br />

Here is the result page for the second scenario, with empty output group 2, 'Unhandled Exception' being dynamically created by the library package to capture any unhandled exceptions, in HTML format:
<img src="/images/2024/08/18/UT-Scenario-2.png">

## 5 Blog Internal Links Hierarchy
[&uarr; Contents](#contents)<br />
[&darr; Markdown Headings Hierarchy](#markdown-headings-hierarchy)<br />
[&darr; Markdown Links to Headings](#markdown-links-to-headings)<br />
[&darr; Example of Links from Level 1 to 4](#example-of-links-from-level-1-to-4)<br />

This series of articles was written using GitHub markdown notation, generated via the Jekyll site generator and served through GitHub pages.

In markdown headings are denoted by starting the line with from 1 to 6 #'s with heading level being the number of #s, and 1 being the top level.

### Markdown Headings Hierarchy
[&uarr; 5 Blog Internal Links Hierarchy](#5-blog-internal-links-hierarchy)<br />

Here is the list of markdown headings from the current article.

```
# Contents
## 1 Installation
### Manual Installation
#### File System Installs
#### Database Installs
### Automated Installation
#### One Step Install
#### Powershell Script Install-Ico.ps1
## 2 Running the Solution Methods
### Sqlplus Scripts
### Powershell Driver Script - Run-All.ps1
## 3 Code Instrumentation
### Code Timing
#### Code Timing Example Output
#### Code Timing Example Code
##### Pop_Table_Iterate Extract
##### insert_Initial_Path
##### insert_Paths Extract
### Execution Plans
#### Statement with Hint
#### insert_Paths Extract
#### Execution Plan - Pop_Table_Iterate: England dataset, 10'th iteration
## 4 Unit Testing
### Input Side
#### Example Wrapper API Function - Array_Iterate
#### Shared JSON File
### Output Side
#### Results Summary for All Solution Methods
#### Unit Test Unit Folders
#### Unit Test Report: Best Item Category Combis - Array_Iterate
#### Scenario 2: Choose r / n (r < n) [Category Set: Choose Range (r / n)]
## 5 Blog Internal Links Hierarchy
### Markdown Headings Hierarchy
### Markdown Links to Headings
### Example of Links from Level 1 to 4
#### Level 1 Links
#### Level 2 Links
#### Level 3 Links
#### Level 4 Links
## 6 Conclusion
```

If we treat each heading as being the child of the previous heading of higher level, if any, then we can see the headings as forming a hierarchy, or tree structure.

### Markdown Links to Headings
[&uarr; 5 Blog Internal Links Hierarchy](#5-blog-internal-links-hierarchy)<br />

Internal links can be included in an article that jump to a heading, say '## Level Two Heading', by using the following text:

```
[Level Two Heading](#level-two-heading)
```

The URL is a single # followed by a modified form of the heading text, with an integer suffix in the event of duplicate headings.

By adding URLs after each heading we can allow for easy navigation up or down one level of the hierarchy. This can be done manually but it's quite a time-consuming process, so we have developed a Powershell script to automate it.

You can see the results throughout the articles, and below we display a screenclip followed by the markdown generated by the script for sections of the current article starting with the level 1 section 'Contents' and drilling down to the level 4 section, 'Results Summary for All Solution Methods'. We use a &uarr; icon as a prefix to indicate a link that goes to the parent heading, and a &darr; icon for the child heading links.

The Powershell script allows for headings to be excluded from the links hierarchy by preceding the #s with a '!' character, which was applied to the subheadings in the Conclusion section below.

### Example of Links from Level 1 to 4
[&uarr; 5 Blog Internal Links Hierarchy](#5-blog-internal-links-hierarchy)<br />
[&darr; Level 1 Links](#level-1-links)<br />
[&darr; Level 2 Links](#level-2-links)<br />
[&darr; Level 3 Links](#level-3-links)<br />
[&darr; Level 4 Links](#level-4-links)<br />

In this section we show the lists of links in the current article starting from the 'Contents' heading and drilling down to a level 4 heading, with screenclips and the markdown code.

#### Level 1 Links
[&uarr; Example of Links from Level 1 to 4](#example-of-links-from-level-1-to-4)<br />

#### Screenclip of Links

This is the top level in the hierarchy so there is no parent link.

<img src="/images/2024/08/18/TOC - Contents.png">

#### Markdown Code for Links

```
# Contents
[&darr; 1 Installation](#1-installation)<br />
[&darr; 2 Running the Solution Methods](#2-running-the-solution-methods)<br />
[&darr; 3 Code Instrumentation](#3-code-instrumentation)<br />
[&darr; 4 Unit Testing](#4-unit-testing)<br />
[&darr; 5 Blog Internal Links Hierarchy](#5-blog-internal-links-hierarchy)<br />
[&darr; 5 Conclusion](#5-conclusion)<br />
```

#### Level 2 Links
[&uarr; Example of Links from Level 1 to 4](#example-of-links-from-level-1-to-4)<br />

#### Screenclip of Links

<img src="/images/2024/08/18/TOC - Unit Testing.png">

#### Markdown Code for Links

```
## 4 Unit Testing
[&uarr; Contents](#contents)<br />
[&darr; Input Side](#input-side)<br />
[&darr; Output Side](#output-side)<br />
```

#### Level 3 Links
[&uarr; Example of Links from Level 1 to 4](#example-of-links-from-level-1-to-4)<br />

#### Screenclip of Links

<img src="/images/2024/08/18/TOC - Output Side.png">

#### Markdown Code for Links

```
### Output Side
[&uarr; 4 Unit Testing](#4-unit-testing)<br />
[&darr; Results Summary for All Solution Methods](#results-summary-for-all-solution-methods)<br />
[&darr; Unit Test Unit Folders](#unit-test-unit-folders)<br />
[&darr; Unit Test Report: Best Item Category Combis - Array_Iterate](#unit-test-report-best-item-category-combis---array_iterate)<br />
[&darr; Scenario 2: Choose r / n (r < n) [Category Set: Choose Range (r / n)]](#scenario-2-choose-r--n-r--n-category-set-choose-range-r--n)<br />
```

#### Level 4 Links
[&uarr; Example of Links from Level 1 to 4](#example-of-links-from-level-1-to-4)<br />

#### Screenclip of Links

This is the bottom level in the hierarchy so there are no child links.

<img src="/images/2024/08/18/TOC - Results Summary for All Solution Methods.png">

#### Markdown Code for Links

```
#### Results Summary for All Solution Methods
[&uarr; Output Side](#output-side)<br />
```
## 6 Conclusion
[&uarr; Contents](#contents)<br />

In this article, we have described the various kinds of automation used in the development, testing and installation of the code and other artefacts for this series of articles.

We can also list here some of the features and concepts considered in the whole series.
- [OPICO 1-8: Optimization Problems with Items and Categories in Oracle](#list-of-articles)

### Sequence Generation
- 4 types of sequence defined
- sequence generation explained via recursion...
 - ...implemented by recursion and by iteration

### Optimization Problems
- sequence truncation using simple maths
- value filtering techniques with approximation and bounding
- two-level iterative refinement methods

### SQL
- recursive SQL
    - materializing subqueries via hints or use of temporary tables
    - cycles and some anomalies
- storing sequences of items in SQL by concatenation, nested tables, and linking tables
- index organised tables
- partitioned outer joins
- splitting concatenated lists into items via row-generation
- combining lists of items into concatenated strings by aggregation
- passing bind variables into views via system contexts
- automated generation of execution plans

### PL/SQL
- PL/SQL with embedded SQL as alternative solution methods to recursive SQL...
- ...with sequence generation by both recursion and iteration, with performance comparisons
- use of arrays and temporary tables for intermediate storage, with performance comparisons
- methods for compact storage of sequences of items
- use of PL/SQL functions in SQL and performance effects of context switching
- automated code timing

### Verification Techniques
- problem abstraction
- maths
- multiple datasets
- multiple solution methods
- perturbation analysis
- unit testing, with [The Math Function Unit Testing Design Pattern](https://brenpatf.github.io/2023/06/05/the-math-function-unit-testing-design-pattern.html)

### Automation
- installation
- running the solution methods
- code instrumentation
- unit testing
- blog internal links hierarchy

<div class="opico-nav" style="border: 1px solid #ccc; padding: 0.8em; margin: 1.5em 0; background-color: #f9f9f9;">
  <strong>📘 OPICO Series:</strong>
  <a href="/2024/06/30/opico-series-index.html">Index</a>
  {% if page.opico_prev %}
    | <a href="{{ page.opico_prev }}">◀ Previous</a>
  {% endif %}
  {% if page.opico_next %}
    | <a href="{{ page.opico_next }}">Next ▶</a>
  {% endif %}
</div>
