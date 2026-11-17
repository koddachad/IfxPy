# Building the Kodda IfxPy wheel

This fork carries Kodda's fixes to IfxPy (see KAN-98). It builds one wheel,
`ifxpy-<version>-cp312-cp312-manylinux2014_x86_64...whl`, which installs on both
Ubuntu 24.04 and RHEL 8.

## Prerequisites

- Docker on the build machine.
- This fork checked out at `~/repos/IfxPy`. `origin` is
  `git@github.com:koddachad/IfxPy.git` and `upstream` is
  `https://github.com/OpenInformix/IfxPy.git`.
- The OneDB ODBC driver headers and libraries. Kodda ships them at
  `kodda-platform/required/linux/ubuntu/onedb-odbc-driver`.

## 1. Set the version

Bump `VERSION` in `IfxPy/setup.py`, using `3.0.5.post2`, `3.0.5.post3` and so on.
A new version number lets pip see each rebuild as a new release.

## 2. Build

```bash
mkdir -p ~/ifxpy-dist && docker run --rm -v ~/repos/IfxPy/IfxPy:/src:ro -v ~/kodda-platform/required/linux/ubuntu/onedb-odbc-driver:/onedb:ro -v ~/ifxpy-dist:/out quay.io/pypa/manylinux_2_28_x86_64 bash -c 'cp -r /src /build && cd /build && export INFORMIXDIR=/onedb CSDK_HOME=/onedb CFLAGS="-Wno-error=incompatible-pointer-types -Wno-error=implicit-function-declaration -Wno-error=int-conversion" && /opt/python/cp312-cp312/bin/python -m pip wheel . --no-deps -w /tmp/raw && auditwheel repair /tmp/raw/*.whl --exclude libthcli.so --exclude libifdmr.so -w /out'
```

The wheel is written to `~/ifxpy-dist/`.

- The `manylinux_2_28` image produces a wheel that runs on both Ubuntu and RHEL 8.
- Without `INFORMIXDIR` and `CSDK_HOME`, `setup.py` tries to download the ODBC
  driver from HCL.
- The image uses GCC 14, which treats some old C constructs in `ifxpyc.c` as
  errors. The `CFLAGS` turn those back into warnings.
- `--exclude libthcli.so --exclude libifdmr.so` keeps the ODBC driver libraries
  out of the wheel. At runtime they come from the OneDB driver in the Kodda image.

## 3. Test in kodda_dev

Use the `kodda_dev` container. `kodda_test` is only for testing prebuilt Kodda
images.

1. Copy the wheel to `~/kodda-platform/tmp/`. The container sees that folder as
   `/usr/local/kodda/tmp/`.
2. Install the wheel into the branch's venv:

   ```bash
   docker exec kodda_dev /usr/local/venv/<branch>/bin/pip install --force-reinstall --no-deps /usr/local/kodda/tmp/<wheel>
   ```

   `--force-reinstall` is only needed if you reuse a version number.
3. Restart the container so the Celery workers load the new driver. Workers that
   are already running keep the old driver until they restart.

   ```bash
   docker restart kodda_dev
   ```

4. Check the KAN-98 behaviour against an Informix database that has transaction
   logging (rollback can only be checked in a logged database):
   - BIGINT values round trip without being cut to 32 bits, and are not read back
     as NULL.
   - `field_type` reports `bigint` for both BIGINT and INT8 columns.
   - In `executemany`, a row the server rejects raises
     `Row N rejected: <server message>`.
   - After that error the batch is rolled back, so no rows are left in the table.
5. Check for memory leaks. Run `executemany` with BIGINT and DECIMAL values many
   times in a loop. The process's memory should stay flat.
6. Run the IfxPy test suite (see `RunTests.md`) against the `ifxpy_test` database.
   Some tests fail on every build because of missing tables (staff, sales and
   others) and a missing sbspace. Compare the results with the previous build
   instead of expecting every test to pass.
7. Reload the Informix mappings in Kodda, then run the Kodda data tests:

   ```bash
   docker exec -w /usr/local/kodda kodda_dev /usr/local/venv/<branch>/bin/python -m pytest testscripts/DataTests
   ```

## 4. Ship

1. Commit and push this fork.
2. In `kodda-platform`, do the following for each of `required/linux/rhel8` and
   `required/linux/ubuntu`:
   - Put the new wheel in `wheelhouse/` and remove the old IfxPy wheel.
   - Update the `IfxPy==` pin in `wheelhouse/requirements.txt`.
   - Update `wheelhouse3.12.zip` from that folder:

     ```bash
     zip wheelhouse3.12.zip wheelhouse/requirements.txt wheelhouse/<new wheel>
     zip -d wheelhouse3.12.zip wheelhouse/<old wheel>
     ```

3. Update the `IfxPy==` pin in the root `requirements.txt`.
4. Commit. The zips are stored in Git LFS.
