build_hotdoc = false

if meson.is_cross_build()
    if get_option('doc').enabled()
        error('Documentation enabled but building the doc while cross building is not supported yet.')
    endif

    message('Documentation not built as building it while cross building is not supported yet.')
    subdir_done()
endif

required_hotdoc_extensions = ['gi-extension', 'c-extension', 'gst-extension']
if gst_dep.type_name() == 'internal'
    gst_proj = subproject('gstreamer')
    plugins_cache_generator = gst_proj.get_variable('plugins_cache_generator')
else
    required_hotdoc_extensions += ['gst-extension']
    plugins_cache_generator = find_program(join_paths(gst_dep.get_pkgconfig_variable('libexecdir'), 'gstreamer-' + api_version, 'gst-plugins-doc-cache-generator'),
        required: false)
endif

plugins_cache = join_paths(meson.current_source_dir(), 'plugins', 'gst_plugins_cache.json')

if plugins.length() == 0
    message('All bad plugins have been disabled')
elif plugins_cache_generator.found()
    plugins_doc_dep = custom_target('bad-plugins-doc-cache',
        command: [plugins_cache_generator, plugins_cache, '@OUTPUT@', '@INPUT@'],
        input: plugins,
        output: 'gst_plugins_cache.json',
    )
else
    warning('GStreamer plugin inspector for documentation not found, can\'t update the cache')
endif

hotdoc_p = find_program('hotdoc', required: get_option('doc'))
if not hotdoc_p.found()
    message('Hotdoc not found, not building the documentation')
    subdir_done()
endif

hotdoc = import('hotdoc')
foreach extension: required_hotdoc_extensions
    if not hotdoc.has_extensions(extension)
        if get_option('doc').enabled()
            error('Documentation enabled but @0@ missing'.format(extension))
        endif

        message('@0@ extensions not found, not building documentation requiring it'.format(extension))
        subdir_done()
    endif
endforeach

if not build_gir
    if get_option('doc').enabled()
        error('Documentation enabled but introspection not built.')
    endif

    message('Introspection not built, can\'t build the documentation')
    subdir_done()
endif

build_hotdoc = true

docconf = configuration_data()
docconf.set('GST_API_VERSION', api_version)

configure_file(input : 'gst_api_version.in',
               output : 'gst_api_version.md',
               configuration : docconf)

root_rel = '..'
excludes = []
foreach f: [
        'gst/frei0r/frei0r.h',
        'gst/mxf/mxful.c',
        'gst-libs/gst/mpegts/gstmpegts-private.h',
        'gst-libs/gst/player/*-private.h',
        'gst-libs/gst/codecparsers/nalutils.h',
        'ext/closedcaption/bcd.h',
        'ext/closedcaption/bit_slicer.[ch]',
        'ext/closedcaption/decoder.[ch]',
        'ext/closedcaption/macros.h',
        'ext/closedcaption/misc.h',
        'ext/closedcaption/raw_decoder.[ch]',
        'ext/closedcaption/sampling_par.[ch]',
        'ext/closedcaption/sliced.[ch]',
        'ext/*/*.h',
        'sys/*/*.h',
        'gst/*/*.h',
        'gst-libs/gst/*/*-prelude.h'
    ]
    excludes += [join_paths(meson.current_source_dir(), root_rel, f)]
endforeach

libs = []
if build_gir
    libs = [
        {'name': 'mpegts', 'gir': mpegts_gir, 'lib': gstmpegts_dep},
        {'name': 'player', 'gir': player_gir, 'lib': gstplayer_dep},
        {'name': 'insertbin', 'gir': insertbin_gir, 'lib': gstinsertbin_dep},
        {'name': 'codecparsers', 'lib': gstcodecparsers_dep},
        {'name': 'adaptivedemux', 'lib': gstadaptivedemux_dep},
        {'name': 'webrtc', 'gir': webrtc_gir, 'lib': gstwebrtc_dep, 'suffix': 'lib'},
        {'name': 'audio', 'gir': audio_gir, 'lib': gstbadaudio_dep, 'prefix': 'bad-'},
        {'name': 'transcoder', 'gir': transcoder_gir, 'lib': gst_transcoder_dep},
        {'name': 'codecs', 'gir': codecs_gir, 'lib': gstcodecs_dep},
   ]
endif

has_gi_extension = hotdoc.has_extensions('gi-extension')
has_c_extension = hotdoc.has_extensions('c-extension')
libs_doc = []
foreach lib_def: libs
  name = lib_def['name']
  lib = lib_def['lib']
    if lib_def.has_key('gir')
        if has_gi_extension
            gir = lib_def['gir']
            prefix = lib_def.get('prefix', '')
            suffix = lib_def.get('suffix', '')
            libs_doc += [hotdoc.generate_doc(prefix + name + suffix,
                project_version: api_version,
                gi_c_sources: [join_paths(root_rel, 'gst-libs/gst', name, '*.[hc]')],
                gi_sources: gir[0].full_path(),
                gi_c_source_filters: excludes,
                sitemap: join_paths('libs', name, 'sitemap.txt'),
                index: join_paths('libs/', name, 'index.md'),
                gi_index: join_paths('libs/', name, 'index.md'),
                gi_smart_index: true,
                gi_order_generated_subpages: true,
                dependencies: [lib, gir],
                c_flags: '-DGST_USE_UNSTABLE_API',
                install: false,
            )]
        endif
    else
        if has_c_extension
            libs_doc += [hotdoc.generate_doc('gst-plugins-bad-' + name,
                sitemap: join_paths('libs', name, 'sitemap.txt'),
                index: join_paths('libs', name, 'index.md'),
                project_version: api_version,
                c_sources: [join_paths(root_rel, 'gst-libs/gst/', name, '*.[hc]')],
                c_source_filters: excludes,
                c_index: join_paths('libs', name, 'index.md'),
                c_smart_index: true,
                c_order_generated_subpages: true,
                extra_c_flags: ' -DGST_USE_UNSTABLE_API',
                dependencies: lib,
                install: false,
            )]
        endif
    endif
endforeach

list_plugin_res = run_command(python3, '-c',
'''
import sys
import json

with open("@0@") as f:
    print(':'.join(json.load(f).keys()), end='')
'''.format(plugins_cache))

assert(list_plugin_res.returncode() == 0,
  'Could not list plugins from @0@'.format(plugins_cache))

plugins_doc = []
foreach plugin_name: list_plugin_res.stdout().split(':')
    plugins_doc += [hotdoc.generate_doc(plugin_name,
        project_version: api_version,
        sitemap: 'plugins/sitemap.txt',
        index: 'plugins/index.md',
        c_flags: '-DGST_USE_UNSTABLE_API',
        gst_index: 'plugins/index.md',
        gst_smart_index: true,
        gst_c_sources: [
            join_paths(root_rel, 'sys/*/*.[ch]'),
            join_paths(root_rel, 'sys/*/*.cpp'),
            join_paths(root_rel, 'ext/*/*.[ch]'),
            join_paths(root_rel, 'ext/*/*.cpp'),
            join_paths(root_rel, 'gst/*/*.[ch]'),
        ],
        dependencies: [gst_dep, plugins],
        install: false,
        disable_incremental_build: true,
        gst_cache_file: plugins_cache,
        gst_c_source_filters: excludes,
        gst_order_generated_subpages: true,
        gst_plugin_name: plugin_name,
        include_paths: join_paths(meson.current_source_dir(), '..'),
    )]
endforeach
