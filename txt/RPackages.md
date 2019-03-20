## <a name="packages"/> R packages

### Check R package installation
A simple test if a package can be loaded can be done by this function:

```SQL
CREATE OR REPLACE FUNCTION R_test_require(fname text)
RETURNS boolean AS
$BODY$
    return(require(fname,character.only=T))
$BODY$
LANGUAGE 'plr';
```

If you want to check for a package called 'rpart', you would do
```SQL
SELECT R_test_require('rpart');
```

And it will return TRUE if the package could be loaded and FALSE if it couldn't. However, this only works on the node that you are currently logged on to.

To test the R installations on all nodes you would first create a dummy table with a series of integers that will be stored on different nodes in GPDB, like this:

```SQL
DROP TABLE IF EXISTS simple_series;
CREATE TABLE simple_series AS (SELECT generate_series(0,1000) AS id);
```

Also, since we want to know which host we are on we create a function to tell us:

```SQL
CREATE OR REPLACE FUNCTION R_return_host()
RETURNS text AS
$BODY$
  return(system("hostname",intern=T))
$BODY$
LANGUAGE 'plr';
```

Now we can check for each id (ids are stored on different nodes) if rpart is installed like this:
```SQL
DROP TABLE IF EXISTS result_nodes;
CREATE TABLE result_nodes AS 
    (SELECT id, R_return_host() AS hostname, R_test_require('rpart') AS result 
    FROM simple_series group by id); 
```

`result_nodes` is a table that contains for every id, the host that it is stored on as hostname, and the result of R_test_require as result. Since we only want to know for every host once, we group by hostname like this:

```SQL
select hostname,bool_and(result) as host_result from result_nodes group by hostname order by hostname;
```

For a hostname where `R_test_require` returned true for all ids, the value in the column `host_result` will be true. If on a certain host the package couldn't be loaded, `host_result` will be false.

### Installing R packages

For a given R library, identify all dependent R libraries and each library’s web url.  This can be found by selecting the given package from the following navigation page: 
https://cran.r-project.org/web/packages/available_packages_by_name.html 

From the page for the `arm` library, it can be seen that this library requires the following R libraries: `Matrix`, `lattice`, `lme4`, `R2WinBUGS`, `coda`, `abind`, `foreign`, `MASS`

From the command line, use wget to download the required libraries' `tar.gz` files to the master node:

```
wget https://cran.r-project.org/src/contrib/arm_1.5-03.tar.gz
wget https://cran.r-project.org/src/contrib/Archive/Matrix/Matrix_1.0-1.tar.gz
wget https://cran.r-project.org/src/contrib/Archive/lattice/lattice_0.19-33.tar.gz
wget https://cran.r-project.org/src/contrib/lme4_0.999375-42.tar.gz
wget https://cran.r-project.org/src/contrib/R2WinBUGS_2.1-18.tar.gz
wget https://cran.r-project.org/src/contrib/coda_0.14-7.tar.gz
wget https://cran.r-project.org/src/contrib/abind_1.4-0.tar.gz
wget https://cran.r-project.org/src/contrib/foreign_0.8-49.tar.gz
wget https://cran.r-project.org/src/contrib/MASS_7.3-17.tar.gz
```

Using `gpscp` and the hostname file, copy the `tar.gz` files to the same directory on all nodes of the Greenplum cluster.  Note that this may require root access. (note: location of host file may be different. On our DCA its /home/gpadmin/all_hosts)

```
gpscp -f /home/gpadmin/hosts_all lattice_0.19-33.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all Matrix_1.0-1.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all abind_1.4-0.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all coda_0.14-7.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all R2WinBUGS_2.1-18.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all lme4_0.999375-42.tar.gz =:/home/gpadmin 
gpscp -f /home/gpadmin/hosts_all MASS_7.3-17.tar.gz =:/home/gpadmin
gpscp -f /home/gpadmin/hosts_all arm_1.5-03.tar.gz =:/home/gpadmin
```

`gpssh` into all segments (`gpssh -f /home/gpadmin/all_hosts`).  Install the packages from the command prompt using the `R CMD INSTALL` command.  Note that this may require root access 

```
R CMD INSTALL lattice_0.19-33.tar.gz Matrix_1.0-1.tar.gz abind_1.4-0.tar.gz coda_0.14-7.tar.gz R2WinBUGS_2.1-18.tar.gz lme4_0.999375-42.tar.gz MASS_7.3-17.tar.gz arm_1.5-03.tar.gz -l $R_HOME/library
```

Check that the newly installed package is listed under the `$R_HOME/library` directory on all the segments (convenient to use `gpssh` here as well). If you do not see it in that directory:
1. Search for the directory (the name of the package) and determine which path it has installed to
2. Copy over the contents of this directory to the `$R_HOME/library` directory

### Package versions
Sometimes the current version of a package has dependencies on an earlier version of R. If this happens, you might get an error message like:

```
In getDependencies(pkgs, dependencies, available, lib) :
  package ‘matrix’ is not available (for R version 2.13.0)
```

Fortunately, there are older versions of most packages available in the CRAN archive. One heuristic we’ve found useful is to look at the release date of the R version installed on the machine. At the time of writing, it is v2.13 on our analytics DCA, which was released on 13-Apr-2011 (https://cran.r-project.org/src/base/R-2/). Armed with this date, go to the archive folder for the package you are installing and find the version that was released immediately prior to that date. For instance, the v1.5.3 of the package `glmnet` was released on 01-Mar-2011 and should be compatible with R v2.13 (https://cran.r-project.org/src/contrib/Archive/glmnet/ )



