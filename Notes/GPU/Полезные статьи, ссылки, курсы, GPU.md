#GPU 
[[Новости, GPU]]

______
#Алгоритмы #Компиляторы 
DIrectXShaderCompiler source code:
https://github.com/microsoft/DirectXShaderCompiler

______
#Nvidia #Cpp

## NVTX

[Documentation](https://docs.nvidia.com/nsight-visual-studio-edition/2020.1/nvtx/index.html)

Библиотека созданная для профилирования кода на GPU

Пример:

```cpp

nvtxRangePushA(__FUNCTION__);
// Some gpu code 
nvtxRangePop();

```

Теперь в NVIDIA System Nsight, NVIDIA Nsight Coimpute будет видна наша функция и как она выполняется

_____