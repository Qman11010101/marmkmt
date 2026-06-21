# marmkmt

`marmkmt` converts the Margrete Plugin SDK v2 C++ ABI into the stable C ABI
defined by [`include/marmkmt.h`](include/marmkmt.h). It builds as one x64
Windows static library; it does not build a bridge DLL.

The SDK header is provided by the `third_party/MargretePluginSDK` Git
submodule, pinned to commit `b8d0c87125e090880a75f6f0ff57a773510299a8`.

Clone with submodules enabled:

```powershell
git clone --recurse-submodules <repository-url>
```

For an existing checkout, initialize the SDK with:

```powershell
git submodule update --init --recursive
```

## Build with Visual Studio 2022

```powershell
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

The static library is generated at `build/Release/marmkmt.lib`. A final plugin
DLL must link this library and provide the `mg_plugin_init` function declared
by the public header. Because no code in the wrapper necessarily references a
symbol from the archive, the final DLL link must retain the archive as a whole
(MSVC: `/WHOLEARCHIVE:marmkmt.lib`) or explicitly export/reference both
`MargretePluginGetInfo` and `MargretePluginCommandCreate`.
