### Performance observation of the file `examples/FCCee/flavour/Bc2TauNu/analysis_stage1.py`

If `analysis_stage1.py` is run without any changes there are around **3600** events processed per second (on 8 cpus).

The largest impact to the elapsed time of the program has:
< 1 >. the function `Minimize()`, which is called from `Algorithms::minimize_thrust::operator()` in file `analyzers/dataframe/Algorithms.cc`.
< 2 >. the function `VertexFitterSimple::VertexFitter_Tk` in file `analyzers/dataframe/VertexFitterSimple.cc`.
< 3 >. the jitting (which is constant)

---

Clarifying the jitting (< 3 >): It takes a fixed constant time, and currently the sandwich timer also count this time. As this time cannot be decreased, it is always subtracted before finding the events processed per second. This is how we get **3600** events processed per second (EP/s). **All reported results exclude jitting.**

---

First inspecting < 1 >. To show how large its impact is, the original body of `Algorithms::minimize_thrust::operator()` is removed and only a vector of zeros is returned. In this manner, we obtain almost **8000** EP/s. We can easily confirm that the problem comes exactly from the `Minimize()` method. In particular, if we keep `Algorithms::minimize_thrust::operator()` as it was originally, but in stead we return a vector of zeros, we are again getting **3600** EP/s. This means that the problem is not coming from some other instance which is using the returned result, but rather the problem comes precisely from the `Minimize()`.

In the original setup, `Minimize()` accounts for above 50% of the work of the program. Moreover, `Algorithms::minimize_thrust::operator()` is being called approx 8000 times per root file. We observe that relaxing the constraints of the minimizer leads to huge improvements. Originally, there is:
```C++
  min->SetMaxFunctionCalls(1000000); // for Minuit/Minuit2 
  min->SetMaxIterations(10000);  // for GSL 
  min->SetTolerance(0.001);
  min->SetPrintLevel(0);
  double step[3] = {0.001,0.001,0.001};
```

Please  note that `min->SetMaxIterations(10000);` is irrelevant for our case, as we are using `Minuit2` in the inspected file (`analysis_stage1.py`).

Small tweaks and their corresponsing new number of processed events per seconds:
```
exclusive change: SetMaxFunctionCalls(750000) --> 4440 EP/s
exclusive change: SetMaxFunctionCalls(500000) --> 5882 EP/s
exclusive change: SetMaxFunctionCalls(100000) --> 7547 EP/s

exclusive change: SetTolerance(0.0012599999) --> 3600 EP/s (no change from original)
exclusive change: SetTolerance(0.00126)      --> 7600 EP/s
exclusive change: SetTolerance(0.0013)       --> 7600 EP/s
exclusive change: SetTolerance(0.0015)       --> 7600 EP/s

exclusive change: step={0.0012,0.0012,0.0012}--> 1089 EP/s (much worse)

change: step={0.0012,0.0012,0.0012} && SetTolerance(0.0012) --> 7766 EP/s
```
If the final results of the analysis allow such an increase in the step size and tolerance size, then the EP/s can be increased by above 100% as shown above.

---

Regarding < 2 >, using a similar technique, as in < 1 >, to verify how large the impact of this function is: we see that it takes slightly more than 20% of the work of the whole program.

Making this function more efficient is very essential as it is being called more than 40 000 times per root file. I can see that your heap is getting unneccessary full during `VertexFitterSimple::VertexFitter_Tk`. 2 important points here:
1. Here, a vector of matrices is defined, and vectors of vectors (ex: ai, Di, Wi, ...), however only a single objects (meaning a single matrix/vector in stead of a vector of matrices/vectors) could have been used and its value being updated in each iteration. The final values of some nested structures are not needed in the return statement of the function, and also that in position i previous indexes are not needed. Therefore, only 1 single object which is being updated could be used. Plus, all structures are now defined in the beginning, but they are not needed anymore in the same scope (except for deallocation). 
2. Using `TVectorD`, `TMatrixD` and so on has expensive creation time as they are derived from `TObject` class, which carries a lot of properties and thus has expensive creation.

As an alternative, do `ROOT::VecOps::RVec<double> fi (Ntr);` in stead of `Double_t *fi = new Double_t[Ntr];`. In this way, no longer need to manually delete objects (`new` is considered bad practice since it is also not handling automatically errorous conditions). Plus, in ROOT v.26 which is coming as a stable release very soon, we are optimizing the work of small `RVec`-s, which should further increase the performance in your case.

Remark is, the while loop inside of `VertexFitterSimple::VertexFitter_Tk` at first glance looks that it might be repeated large number of times, but that is not the case. This loop:
```
  Int_t Ntry = 0;
  Int_t TryMax = 100;
  if (BeamSpotConstraint) TryMax = TryMax * 5;
  Double_t eps = 1.0e-9; // vertex stability
  Double_t epsi = 1000.;
  //

  while (epsi > eps && Ntry < TryMax)
```
It is on average iterated 5 number of times per call of `VertexFitterSimple::VertexFitter_Tk`, so the stopping conditions are fine. The problem is the inefficient memory allocation in and outside the function as already discussed.

---

As a conclusion, tuning the constraints of the minimizer should be applied if possible. Allocation of objects on the heap can be done better. And consider using RVec-s in stead of TObjects.

And also as previously discussed avoid `(at(i))` to index vectors and call functions by passing large objects by reference. 
