In order to produce more detailed report build the project with:
```sh
source ./setup.sh
mkdir build install
cd build
cmake .. -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_INSTALL_PREFIX=../install
make install
cd ..
```
As before, if a modification in `analyzers/dataframe/` is done, it is sufficient to only `make install`.

In the python file under inspection do:
```python
#ROOT.gErrorIgnoreLevel = ROOT.kFatal
verbosity = ROOT.Experimental.RLogScopedVerbosity(ROOT.Detail.RDF.RDFLogChannel(), ROOT.Experimental.ELogLevel.kInfo);
```

In this way, it is now possible to read the C++ Internal loop, which is doing the computations. It is also possible to read the time needed for jitting.

Currently, the "sandwich" clock also includes jitting:
```python
start_time = time.time()
...
elapsed_time = time.time() - start_time
```

For more advanced analysis consider using `perf` and [Flamegraph](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html).
