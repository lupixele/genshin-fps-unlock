ZIG ?= zig
BUILD_DIR := .build

release : unlockfps_nc/bin/ReleaseMin/net8.0-windows/win-x64/publish/unlockfps_nc.exe

# The stub used to be built with MSVC only (see UnlockerStub/UnlockerStub.vcxproj).
# zig ships clang plus libc++, so the same sources build on any machine without
# Visual Studio. Keep using the vcxproj if you prefer MSVC.
UnlockerStub/UnlockerStub.dll : $(wildcard UnlockerStub/*.cpp UnlockerStub/*.c UnlockerStub/*.h)
	mkdir -p $(BUILD_DIR)
	$(ZIG) cc -target x86_64-windows-gnu -O2 -c UnlockerStub/Zydis.c -o $(BUILD_DIR)/zydis.o
	$(ZIG) c++ -target x86_64-windows-gnu -shared -O2 -std=c++20 \
		-o $@ UnlockerStub/dllmain.cpp UnlockerStub/Utils.cpp $(BUILD_DIR)/zydis.o -ldbghelp -lntdll

unlockfps_nc/Resources/%.dll : UnlockerStub/%.dll
	cp $^ $@

unlockfps_nc/bin/%/net8.0-windows/win-x64/publish/unlockfps_nc.exe : unlockfps_nc/unlockfps_nc.csproj unlockfps_nc/Resources/UnlockerStub.dll
	dotnet publish $< -c $* -r win-x64 -f net8.0-windows --self-contained false -p:PublishSingleFile=true -p:PublishReadyToRun=false

clean :
	rm -rf UnlockerStub/UnlockerStub.dll unlockfps_nc/Resources/UnlockerStub.dll unlockfps_nc/bin unlockfps_nc/obj $(BUILD_DIR)
