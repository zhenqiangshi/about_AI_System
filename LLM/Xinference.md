Xinference 在 Linux, Windows, MacOS 上都可以通过 `pip` 来安装。如果需要使用 Xinference 进行模型推理，可以根据不同的模型指定不同的引擎。

如果你希望能够推理所有支持的模型，可以用以下命令安装所有需要的依赖:

pip install "xinference[all]"



注意事项：
1、Xinference 会为每个模型建立独立的环境，这个环境与当前部署的环境（Xinfrence启动）的环境不一致。当然也可以让它使用部署环境。

![[Pasted image 20260709170127.png]]

2、注意模型虚拟空间的cuda版本-torch-vllm的版本


3、注意副本，当前的副本为1，意思是启动一个服务实例。