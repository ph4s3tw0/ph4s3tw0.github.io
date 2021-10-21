# Compiling LLVM with visual studio.
Here at Phasetw0 we hate monolitic compilers. Thats why we exclusively use the llvm compiler infrastructure as our build toolchain.  
Written by 11philip22.  
![index](https://user-images.githubusercontent.com/26529935/138117417-a709eed3-5c86-4b46-9167-e67741684dd2.png)
## Step #1: Build
### Prerequisites
Install both clang extensions in Visual Studio.  
![Screenshot_2021-10-20_16-09-35](https://user-images.githubusercontent.com/26529935/138109331-99c6713e-4c0c-4347-91b3-05b5c7a29c92.png)  

Make sure python3.6+ is installed.  
![Screenshot_2021-10-20_16-07-55](https://user-images.githubusercontent.com/26529935/138109001-80330d36-12c0-452a-a677-4f7f9a6d9def.png)  

Install the `psutil` python module
```
pip install psutil
```
### Compile LLVM
Open the Visual Studio developer console and use the commands bellow to generate a visual studio solution to compile llvm.  
Please also checkout all availbe cmake flags at: https://llvm.org/docs/CMake.html#options-and-variables.
```
git clone https://github.com/llvm/llvm-project.git llvm
cd llvm
cmake -S llvm -B build -DLLVM_ENABLE_PROJECTS="clang;lld" -DLLVM_USE_LINKER=lld -DLLVM_TARGETS_TO_BUILD=X86 -DCMAKE_INSTALL_PREFIX="C:\llvm" -Thost=x64
```
This creates the Visual Studio solution in `./llvm/build`.
![Screenshot_2021-10-20_16-23-54](https://user-images.githubusercontent.com/26529935/138112054-e3514c08-4ed3-4493-ba38-e8dfe0d9b990.png)  

Since compilation will take a really long time i suggest to set the solution's configuration to `Release`.  
Now build the `ALL_BUILD` project to compile llvm.  
![Screenshot_2021-10-20_16-26-23 <](https://user-images.githubusercontent.com/26529935/138112614-2a7f935d-d167-44da-990b-bbf6b15c0de5.png) ![Screenshot_2021-10-21_09-52-07](https://user-images.githubusercontent.com/26529935/138235102-6c363e17-bba6-45fc-ac81-9adf2a055ef5.png)  
Run the `INSTALL` project to install llvm to `C:\llvm`

## Step #2: Using llvm with Visual Studio
First we create a new Visual Studio project with the name llvmTest.  
Create a `HelloWorld.c` source file with the following contents:
```c
#include <stdio.h>
#include <string.h>

char str1[20] = "I love ";
char str2[12] = "11philip22\n";

int main() {
	int x = 1;
	int y = 2;
	printf("hello world %d\n", (x + y) * (x + y));
	char* newStr = _strdup(str1);
	strcat_s(newStr, 20, str2);
	printf("%s", newStr);

	return 0;
}

```
In the project property page set `Platform Toolset` to `LLVM (clang-cl)`.  
![Screenshot_2021-10-21_11-17-16](https://user-images.githubusercontent.com/26529935/138248877-c986b410-a177-458a-9333-a2576573c816.png)
To use our newly build toolchain create a file inside any Visual Studio solution or project called `Directory.build.props` with the contents:
```xml
<Project>
  <PropertyGroup>
    <LLVMInstallDir>C:\llvm</LLVMInstallDir>
	<LLVMToolsVersion>14.0.0</LLVMToolsVersion>
  </PropertyGroup>
</Project>
```
Now Build and Run the project like you would usually do in visual studio.
## Reverences
- https://llvm.org/docs/GettingStartedVS.html
- https://llvm.org/docs/CMake.html
- https://docs.microsoft.com/en-us/cpp/build/clang-support-msbuild?view=msvc-160