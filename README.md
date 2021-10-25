# APRED

APRED (Analysis Platform for Risk, Resilience and Expenditure in Disasters) will provide historical insights from the 2011 disaster data, 
as well as a predictive capability that will be used to improve future funding decisions by local, state and federal agencies.

This repository contains all the source code used to run https://ctil.iu.edu/projects/apred/ 
It mainly consists of 1) backend ETL scripts to load and transform data from StatsAmerica DB, and
2) VueJS frontend to show users the various data aggregated by 1).

## Architecture

<img src="https://docs.google.com/drawings/d/e/2PACX-1vQQ-32ru9jQyRephmCwxx4dVN3DmavPhblELL5pi-yh2AtpFbe9Mf4p4IFd7XsNXJADdNXb9bZnLqOO/pub?w=1440&amp;h=1080">

apred VM runs on IU Jetstream (m1.medium).

```
openstack server create \
    --security-group web \
    --security-group global-ssh \
    --key-name home \
    --flavor m1.medium \
    --image "JS-API-Featured-Ubuntu20-Latest" \
    --nic net-id=ctil \
    apred 

```

Both ETL and auth services currently run on a single apred VM, but they can be launched on 
multiple VMs. We could also run multiple instances of each service as well. 

## ETL Services

We have 2 sets of ETL services we run. 

### `backend/bin/once.sh`

`once.sh` script need to run once per installation and it fetches static(historica) data that do not change

```
./extract/fips.js
./extract/county_geojson.js
./extract/statsamerica_bea_gdp.js
./extract/statsamerica_noaa_storms_past.js
./extract/statsamerica_fema_disasters_past.js

./transform/find_nearest_tribes.js
./transform/count_noaa_storms.js
```

### `backend/bin/daily.sh`

`daily.sh` runs once a day to pull new data from StatsAmerica DB.

```
#APRED raw data extract (all quick)
./extract/fips.js
./extract/statsamerica_eda2018.js
./extract/statsamerica_fema_disasters.js
./extract/statsamerica_noaa_storms.js
./extract/statsamerica_dr.js
./extract/county_geojson.js
./extract/statsamerica_acs.js
./extract/statsamerica_bvi.js

./transform/eda2018.js
./transform/cutters.js
./transform/count_noaa_storms.js
./transform/geojson.js

./load/counties.js
```

The last script `counties.js` generates per-county json to be loaded by APRED UI's county detail page.

Please see each script for more details.

### Frontend

To install the UI, simply git clone this repo, then run `cd ui && npm run build` to build the deployment directory. The deployment directory can then be exposed by any static web servers (Apache, IIS, Nginx, etc..). You might need to adjust the content of `ui/vue.config.js` to set the correct base path.

Th current production instance of the UI (https://ctil.iu.edu/projects/apred) is hosted by [IU sitehost](https://kb.iu.edu/d/axnv) under ctil group. 

#### Accessing IU sitehost / ctil / apred deployment directory

Once you login to ssh.sitehost.iu.edu and `become ctil`, you can access the deployment directory at `/groups/ctil/web/projects/apred`. The sitehost server currently runs a cron job to automatically update the content of the APRED UI if there is any update made to the apred github repo. The update script is located at `~/git/update_apred.sh` relative to the ctil group user directory. The following is the content of this script.

```bash
#!/bin/bash

set -e

cd apred/ui
git reset --hard 2>&1 >/dev/null
new=$(git pull)
if [ ! "$new" == "Already up to date." ]
then
        echo $new
        npm i
        npm run build
        rsync -av dist/ /groups/ctil/web/projects/apred
fi

```

In order to *move* the UI to another webserver, you can simply run another instance of the APRED UI on another webserver, and point to the same backend hosted at IBRC. The existing UI can simply removed (and setup a redirct the new URL?) 



