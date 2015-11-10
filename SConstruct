# -*- Python -*-
import os
import sys
import os.path
import distutils.sysconfig

# A bunch of utility functions to make the rest of the SConstruct file a
# little simpler.

def die(msg):
    sys.stderr.write("ERROR " + msg + "\n")
    Exit(1)

def option(name, dflt):
    result = (ARGUMENTS.get(name) or os.environ.get(name, dflt))
    if type(dflt)==int: result = int(result)
    return result

def findonpath(fname, path):
    for dir in path:
        if os.path.exists(os.path.join(dir, fname)):
            return dir
    raise die("%s: not found" % fname)

def protoc(target, source, env):
    os.system("protoc %s --cpp_out=." % source[0])

def protoemitter(target, source, env):
    for s in source:
        base, _ = os.path.splitext(str(s))
        target.extend([base + ".pb.cc", base + ".pb.h"])
    return target, source

protoc_builder = Builder(action=protoc,
                         emitter=protoemitter,
                         src_suffix=".proto")

# CLSTM requires C++11, and installes in /usr/local by default

prefix = option('prefix', "/usr/local")

env = Environment()
env.Append(CPPDEFINES={'THROW': 'throw', 'CATCH': 'catch', 'TRY': 'try'})
env["BUILDERS"]["Protoc"] = protoc_builder

if option("double", 0):
    env.Append(CPPDEFINES={'LSTM_DOUBLE': '1'})

if option("threads", 0):
  env.Append(CPPDEFINES={'EIGEN_USE_THREADS': '1'})
  env.Append(LIBS=["pthread"])

# With omp=1 support, Eigen and other parts of the code may use
# multi-threading.

if option("omp", 0):
    env["CXX"] = option("CXX", "g++") + \
        " --std=c++11 -Wno-unused-result -fopenmp"
else:
    env["CXX"] = option("CXX", "g++") + " --std=c++11 -Wno-unused-result"

# With profile=1, the code will be compiled suitable for profiling and debug.
# With debug=1, the code will be compiled suitable for debugging.

profile = option("profile", 0)
debug = option("debug", 0)

if profile>1:
    env.Append(CXXFLAGS="-g -pg -fno-inline".split())
    env.Append(CCFLAGS="-g -pg -fno-inline".split())
    env.Append(LINKFLAGS="-g -pg".split())
elif profile>0:
    env.Append(CXXFLAGS="-g -pg -O2".split())
    env.Append(CCFLAGS="-g -pg -O2".split())
    env.Append(LINKFLAGS="-g -pg".split())
elif debug>1:
    env.Append(CXXFLAGS="-g -fno-inline".split())
    env.Append(CCFLAGS="-g".split())
    env.Append(LINKFLAGS="-g".split())
elif debug>0:
    env.Append(CXXFLAGS="-g".split())
    env.Append(CCFLAGS="-g".split())
    env.Append(LINKFLAGS="-g".split())
elif debug<0:
    env.Append(CXXFLAGS="-g -O4 -DNDEBUG -finline".split())
    env.Append(CCFLAGS="-g".split())
else:
    env.Append(CXXFLAGS="-g -O3".split())
    env.Append(CCFLAGS="-g".split())

# Hack for linking some libraries statically.

abslibs = """/usr/lib/x86_64-linux-gnu/libpng12.a
/usr/lib/x86_64-linux-gnu/libprotobuf.a
/usr/lib/x86_64-linux-gnu/libz.a"""

if not option("static", 0):
    env.Append(LIBS=["png", "protobuf"])
else:
    abslibs = [File(x) for x in abslibs.split()]
    env.Append(LIBS=abslibs)

# Extra layers (old layers or testing)

if option("extras", 0):
    env.Append(CPPDEFINES={'CLSTM_EXTRAS': 1})

# Try to locate the Eigen include files (they are in different locations
# on different systems); you can specify an include path for Eigen with
# `eigen=/mypath/include`

if option("eigen", "") == "":
    inc = findonpath("Eigen/Eigen", """
        /usr/include
        /usr/local/include/eigen3
        /usr/include/eigen3""".split())
else:
    inc = findonpath("Eigen/Eigen", [option("eigen")])
env.Append(CPPPATH=[inc])

# You can enable display debugging with `display=1`

if option("display", 0):
    env.Append(LIBS=["zmqpp", "zmq"])
    env.Append(CPPDEFINES={'add_raw': option("add_raw", 'add')})
else:
    env.Append(CPPDEFINES={'NODISPLAY': 1})

# We need to compile the protocol buffer definition as part of the build.

env.Protoc("clstm.proto")

cuda = env.Object("clstm_compute_cuda.o", "clstm_compute_cuda.cc",
           CXX="/usr/local/cuda/bin/nvcc -arch=sm_35 -std=c++11 -x cu -DEIGEN_USE_GPU --expt-relaxed-constexpr",
           CXXCOM=env["CXXCOM"]+" 2>&1 | grep error")


# Build the CLSTM library.

libsrc = ["clstm.cc", "ctc.cc", "clstm_proto.cc", "clstm_prefab.cc",
          "tensor.cc", "batches.cc", "extras.cc", "clstm.pb.cc", 
          "clstm_compute.cc"]
if option("gpu", 1):
  env.Append(LIBS=["cudart","cublas","cuda"])
  env.Append(LIBPATH=["/usr/local/cuda/lib64"])
  env.Append(CPPPATH=["/usr/local/cuda/include"])
  env.Append(CPPDEFINES={'CLSTM_GPU' : 1})
  libsrc = [cuda] + libsrc

libs = env["LIBS"]
libclstm = env.StaticLibrary("clstm", source = libsrc)

all = [libclstm]

programs = """clstmfilter clstmfiltertrain clstmocr clstmocrtrain""".split(
)
for program in programs:
    all += [env.Program(program, [program + ".cc"], LIBS=[libclstm] + libs)]
    Default(program)

# env.Program("fstfun", "fstfun.cc", LIBS=[libclstm]+libs+["fst","dl"])

Alias('install-lib',
      Install(os.path.join(prefix, "lib"), libclstm))
Alias('install-include',
      Install(os.path.join(prefix, "include"), ["clstm.h"]))
Alias('install',
      ['install-lib', 'install-include'])

# If you have HDF5 installed, set hdf5lib=hdf5_serial (or something like that)
# and you will get a bunch of command line programs that can be trained from
# HDF5 data files. This code is messy and may get deprecated eventually.

if option("hdf5lib", "") != "":
    h5env = env.Clone()
    inc = findonpath("hdf5.h", """
        /usr/include
        /usr/local/include/hdf5/serial
        /usr/local/include/hdf5
        /usr/include/hdf5/serial
        /usr/include/hdf5""".split())
    h5env.Append(CPPPATH=[inc])
    h5env.Append(LIBS=["hdf5_cpp"])
    h5env.Append(LIBS=[option("hdf5lib", "hdf5_serial")])
    h5env.Prepend(LIBS=[libclstm])
    for program in "clstmctc clstmseq clstmconv".split():
        h5env.Program(program, [program + ".cc"])

# A simple test of the C++ LSTM implementation.
all += [env.Program("test-lstm", ["test-lstm.cc"], LIBS=[libclstm] + libs)]
all += [env.Program("test-deriv", ["test-deriv.cc"], LIBS=[libclstm] + libs)]
all += [env.Program("test-cderiv", ["test-cderiv.cc"], LIBS=[libclstm] + libs)]
all += [env.Program("test-ctc", ["test-ctc.cc"], LIBS=[libclstm] + libs)]
all += [env.Program("test-timing", ["test-timing.cc"], LIBS=[libclstm] + libs)]

# You can construct the Python extension from scons using the `pyswig` target; however,
# the recommended way of compiling it is with "python setup.py build"

swigenv = env.Clone(SWIGFLAGS=["-python", "-c++"], SHLIBPREFIX="")
swigenv.Append(CPPPATH=[distutils.sysconfig.get_python_inc()])
pyswig = swigenv.SharedLibrary("_clstm.so",
                               ["clstm.i", "clstm.cc", "clstm_proto.cc", "extras.cc",
                                "clstm.pb.cc", "clstm_compute.cc",
                               "clstm_prefab.cc", "ctc.cc"],
                               LIBS=libs)
Alias('pyswig', [pyswig])

destlib = distutils.sysconfig.get_config_var("DESTLIB")
Alias('pyinstall',
      Install(os.path.join(destlib, "site-packages"),
              ["_clstm.so", "clstm.py"]))

Alias('all', [all])
