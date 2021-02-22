project('recFMM', ['c', 'fortran'],
        version : '0.1.1',
        license : 'GPL-3.0-or-later',
        meson_version : '>=0.49.0',
        default_options : ['c_std=gnu99',
                           'buildtype=release',
                           'default_library=both',
                           'prefix=' + meson.current_source_dir()])

vso = '0'

# ========== compiler options and dependencies

cc = meson.get_compiler('c')
fc = meson.get_compiler('fortran')

cc_is_icc = cc.get_id().contains('intel')
use_opencilk = not cc_is_icc and cc.has_argument('-fopencilk')
use_cilkplus = not use_opencilk \
               and (cc_is_icc or cc.has_argument('-fcilkplus'))

# HACK GNU gcc 8.x and 9.x still accept `-fcilkplus` even though they do not
# actually support Cilk...
cc_is_gcc_with_cilk = cc.get_id() == 'gcc' and \
                      cc.version().version_compare('>=5.0.0') and \
                      cc.version().version_compare('<8.1.0')
if use_cilkplus and cc.get_id() == 'gcc' and not cc_is_gcc_with_cilk
  warning('The specified GCC version does not support Cilk' +
          ' (even though -fcilkplus does not trigger an error).')
  use_cilkplus = false
endif

if use_opencilk or use_cilkplus
  have_cilk_headers = cc.has_header('cilk/cilk.h') and \
                      cc.has_header('cilk/cilk_api.h') and \
                      cc.has_header('cilk/reducer_min.h') and \
                      cc.has_header('cilk/reducer_max.h')
  if not have_cilk_headers
    warning('Required Cilk headers not found. Switching to sequential build.')
  endif
endif

if cc_is_icc
  add_project_arguments(['-DICC'], language : 'c') # enables SIMD reduction
endif

# ========== Cilk support

if (use_opencilk or use_cilkplus) and have_cilk_headers
  if use_opencilk
    cilk_cflags  = ['-fopencilk']
    cilk_ldflags = ['-fopencilk']
  elif use_cilkplus and not cc_is_icc
    cilk_cflags  = ['-fcilkplus']
    cilk_ldflags = ['-lcilkrts']
  else
    cilk_cflags  = []
    cilk_ldflags = []
  endif
  add_project_arguments([cilk_cflags, '-DUSING_CILK'], language : 'c')
  add_project_link_arguments([cilk_ldflags], language : 'c')
else
  warning('Could not verify Cilk support with the specified compiler.' +
          ' Cilk Keywords elided.')
endif

# ========== dependencies

# Intel TBB scalable allocator
if get_option('enable_tbbmalloc')
  tbb_dir = get_option('tbb_dir')
  if tbb_dir == ''
    tbb_dir = []
  endif
  tbbmalloc_dep = cc.find_library('tbbmalloc', dirs : tbb_dir, required : false)
  if tbbmalloc_dep.found()
    add_project_arguments(['-DUSING_TBB'], language : 'c')
  else
    warning('TBB scalable allocator option enabled' +
            ' but could not find `tbbmalloc` library. Skipping...')
  endif
else
  tbbmalloc_dep = dependency('', required : false)
endif

# Math library
m_dep = cc.find_library('m', required : false)

# dependency list
recfmm_dep = [tbbmalloc_dep, m_dep]

# ========== instrumentation tools

c_args_san      = []
link_args_san   = []
str_san         = ''
c_args_cs       = []
link_args_cs    = []
str_cs          = ''
c_args_bench    = []
link_args_bench = []
str_bench       = ''

if get_option('cilktool') == 'cilksan' # ----- Cilksan
  if not (use_opencilk and have_cilk_headers)
    warning('Cilksan instrumentation specified but not using OpenCilk compiler;' +
            ' skipping...')
  else
    c_args_san    = ['-fsanitize=cilk','-fno-stripmine',
                     '-fno-vectorize','-fno-unroll-loops']
    link_args_san = ['-fsanitize=cilk']
    str_san       = '_cilksan'
    # FIXME [2021-01-25] Meson does not support static-library-only arguments to
    # build targets defined via `library()`.  See also:
    # https://github.com/mesonbuild/meson/issues/3304
    # REVIEW [2021-01-25] The `-shared-libasan` flag may need to be removed
    # before OpenCilk 1.0 is released
    if get_option('default_library') != 'static'
      link_args_san += ['-shared-libasan']
    endif
  endif
endif

if get_option('cilktool') == 'cilkscale' # ----- Cilkscale
  if not (use_opencilk and have_cilk_headers)
    warning('Cilkscale instrumentation specified but not using OpenCilk compiler;' +
            ' skipping...')
  else
    c_args_cs       = ['-fcilktool=cilkscale','-DCILKSCALE']
    link_args_cs    = ['-fcilktool=cilkscale']
    str_cs          = '_cilkscale'
    c_args_bench    = ['-fcilktool=cilkscale-benchmark','-DCILKSCALE']
    link_args_bench = ['-fcilktool=cilkscale-benchmark']
    str_bench       = str_cs + '_bench'
  endif
endif

# ========== build targets

# headers
subdir('include')
install_headers(recfmm_hdrs, install_dir : 'include')

# source files
subdir('src')
subdir('test')
subdir('example')

def_lap = ['-DLAPLACE']
def_yuk = ['-DYUKAWA']

# recFMM libraries: Laplace & Yukawa
lib_lap = library('adap_laplace' + str_san + str_cs, [src_recfmm_laplace],
                  c_args : def_lap + c_args_san + c_args_cs,
                  link_args : link_args_san + link_args_cs,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_yuk = library('adap_yukawa' + str_san + str_cs, [src_recfmm_yukawa],
                  c_args : def_yuk + c_args_san + c_args_cs,
                  link_args : link_args_san + link_args_cs,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)

# test executables
exe_test_lap = executable('recfmm-test-laplace' + str_san + str_cs, [src_recfmm_test],
                          c_args : def_lap + c_args_san + c_args_cs,
                          link_args : link_args_san + link_args_cs,
                          link_with : lib_lap,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_yuk = executable('recfmm-test-yukawa' + str_san + str_cs, [src_recfmm_test],
                          c_args : def_yuk + c_args_san + c_args_cs,
                          link_args : link_args_san + link_args_cs,
                          link_with : lib_yuk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')

# example executable
exe_example_lap = executable('recfmm-example' + str_san + str_cs, [src_recfmm_example],
                             c_args : def_lap + c_args_san + c_args_cs,
                             link_args : link_args_san + link_args_cs,
                             link_with : lib_lap,
                             dependencies : recfmm_dep,
                             include_directories : [recfmm_include_dir],
                             install_rpath : get_option('prefix') / 'lib',
                             install : true, install_dir : 'example')

# ----- Cilkscale benchmark targets

if c_args_bench != []
  lib_lap_b = library('adap_laplace' + str_bench, [src_recfmm_laplace],
                      c_args : def_lap + c_args_bench,
                      link_args : link_args_bench,
                      dependencies : recfmm_dep,
                      include_directories : [recfmm_include_dir],
                      install : true, install_dir : 'lib',
                      soversion : vso)
  lib_yuk_b = library('adap_yukawa' + str_bench, [src_recfmm_yukawa],
                      c_args : def_yuk + c_args_bench,
                      link_args : link_args_bench,
                      dependencies : recfmm_dep,
                      include_directories : [recfmm_include_dir],
                      install : true, install_dir : 'lib',
                      soversion : vso)
  exe_test_lap_b = executable('recfmm-test-laplace' + str_bench, [src_recfmm_test],
                              c_args : def_lap + c_args_bench,
                              link_args : link_args_bench,
                              link_with : lib_lap,
                              dependencies : recfmm_dep,
                              include_directories : [recfmm_include_dir],
                              install_rpath : get_option('prefix') / 'lib',
                              install : true, install_dir : 'test')
  exe_test_yuk_b = executable('recfmm-test-yukawa' + str_bench, [src_recfmm_test],
                              c_args : def_yuk + c_args_bench,
                              link_args : link_args_bench,
                              link_with : lib_yuk,
                              dependencies : recfmm_dep,
                              include_directories : [recfmm_include_dir],
                              install_rpath : get_option('prefix') / 'lib',
                              install : true, install_dir : 'test')
  exe_example_lap_b = executable('recfmm-example' + str_bench, [src_recfmm_example],
                                 c_args : def_lap + c_args_bench,
                                 link_args : link_args_bench,
                                 link_with : lib_lap,
                                 dependencies : recfmm_dep,
                                 include_directories : [recfmm_include_dir],
                                 install_rpath : get_option('prefix') / 'lib',
                                 install : true, install_dir : 'example')
endif

# ========== tests

test_verifier = find_program('test/verify_test_pass.sh')

test('Laplace [box, acc=3]', test_verifier,
     args : [exe_test_lap, '-d1', '-a3'],
     suite : ['laplace', 'box', 'low_acc'])
test('Laplace [sphere, acc=3]', test_verifier,
     args : [exe_test_lap, '-d2', '-a3'],
     suite : ['laplace', 'sphere', 'low_acc'])
test('Laplace [box, acc=6]', test_verifier,
     args : [exe_test_lap, '-d1', '-a6'],
     suite : ['laplace', 'box', 'high_acc'])
test('Laplace [sphere, acc=6]', test_verifier,
     args : [exe_test_lap, '-d2', '-a6'],
     suite : ['laplace', 'sphere', 'high_acc'])

test('Yukawa [box, acc=3]', test_verifier,
     args : [exe_test_yuk, '-d1', '-a3'],
     suite : ['yukawa', 'box', 'low_acc'])
test('Yukawa [sphere, acc=3]', test_verifier,
     args : [exe_test_yuk, '-d2', '-a3'],
     suite : ['yukawa', 'sphere', 'low_acc'])
test('Yukawa [box, acc=6]', test_verifier,
     args : [exe_test_yuk, '-d1', '-a6'],
     suite : ['yukawa', 'box', 'high_acc'])
test('Yukawa [sphere, acc=6]', test_verifier,
     args : [exe_test_yuk, '-d2', '-a6'],
     suite : ['yukawa', 'sphere', 'high_acc'])
