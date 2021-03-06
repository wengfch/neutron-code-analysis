# 建立 Neutron 的开发环境

本页介绍如何设置可用于在Ubuntu，Fedora或Mac OS X上开发Neutron的Python开发环境。这些说明假定您已经熟悉Git和Gerrit，Git和Gerrit是代码存储库镜像和代码审查工具集 但是，如果您不是，请参阅此Git教程，介绍[如何使用Git](http://git-scm.com/book/en/Getting-Started)和[本指南](http://docs.openstack.org/infra/manual/developers.html#development-workflow)介绍如何使用Gerrit和Git对OpenStack项目的代码贡献。

按照这些说明将允许您运行Neutron的单元测试。 如果您想要在完整的OpenStack环境中运行Neutron，可以使用优秀的DevStack项目来实现。 有一个wiki页面，描述[使用DevStack设置Neutron](https://wiki.openstack.org/wiki/NeutronDevstack)。

## 获取代码

```
git clone git://git.openstack.org/openstack/neutron.git
cd neutron
```

# 测试 Neutron

## 你应该关心什么

有两种方法来测试：

1. 编写单元测试是因为它们需要将补丁合并。这通常包含模拟重测试，断言您的代码像写的一样工作。

2. 尽可能多地考虑您的测试策略，如同您对其余代码一样。 适当使用不同层次的测试来提供高质量的覆盖。 你接触一个agent吗？ 根据实际系统测试！ 你在添加一个新的API吗？ 根据真实的数据库测试竞争条件！ 您是否添加了一个新的cross-cutting功能？ 测试它在真正的云上运行时到底做了什么。

您是否需要手动验证您的更改？如果是这样，接下来的几个部分将试图指导您完成Neutron的不同测试基础架构，以帮助您做出明智的决策，并最好地利用Neutron的测试产品。

## 定义

我们将讨论三类测试：单元，功能和集成。每个类别通常都针对较大的代码范围。除了广泛的分类之外，这里还有一些特点：

* 单元测试 - 应该能够在笔记本电脑上运行，直接跟踪项目的“git clone”。底层系统不能被突变，mock可以用来实现这一点。单元测试通常以功能或类为目标。

* 功能测试 - 针对预配置的环境运行（tools/configure_for_func_testing.sh）。通常测试一个组件，不使用mock代理。

* 集成测试 - 针对运行的云进行，通常针对API级别，而且还针对“场景”或“用户故事”。您可以在*tests/tempest/api*，*tests/tempest/scenario*，*test/fullstack*以及Tempest和Rally项目中找到这样的测试。

Neuron树中的测试通常由所使用的测试基础组织，而不是测试范围。 例如，'unit'目录下的许多测试调用一个API调用，并声明收到了预期的输出。 这种测试的范围是整个Neutron服务器堆栈，并且显然不是典型的单元测试中的特定的功能。

### 测试框架

单元测试（neutron/tests/unit）意图覆盖尽可能多的代码。 它们旨在测试中子树的各种部分，以确保任何新的更改不会破坏现有功能。 单元测试没有要求，也不会对其运行的系统进行更改。 他们使用内存中的sqlite数据库来测试数据库交互。

在每次测试运行开始时：

* RPC listeners are mocked away.
* 假的 Oslo 消息驱动被使用.

在每次测试运行结束时：

* mocks 自动恢复。

* 内存中的数据库被清除内容，但其模式会被维护。

* 全局的Oslo配置对象被重置。

单元测试框架可用于有效测试数据库交互，例如，分布式路由器为运行OVS代理的每个主机分配一个MAC地址。 DVR的DB混合器之一实现了一种列出所有主机MAC地址的方法。 它的测试如下所示：

```
def test_get_dvr_mac_address_list(self):
    self._create_dvr_mac_entry('host_1', 'mac_1')
    self._create_dvr_mac_entry('host_2', 'mac_2')
    mac_list = self.mixin.get_dvr_mac_address_list(self.ctx)
    self.assertEqual(2, len(mac_list))
```

它插入两个新的主机MAC地址，调用被测方法并断言其输出。测试有很多事情要做：

* 它正确地针对被测方法，而不是比所需的更大的范围。

* 它不使用mocks来声明方法被调用，它只是调用该方法并声明其输出（在这种情况下，list方法返回两个记录）。

这是允许的，该方法是建立可测试的 - 该方法具有清晰的输入和输出，没有副作用。

您可以通过将`OS_TEST_DBAPI_ADMIN_CONNECTION`设置为基于文件的URL来获取oslo.db来生成基于文件的sqlite数据库，如本邮件列表中所述。 该文件将被创建，但（混淆）不会是用于数据库的实际文件。 要查找实际文件，请在测试方法中设置一个断点，并检查`self.engine.url`。

```
$ OS_TEST_DBAPI_ADMIN_CONNECTION=sqlite:///sqlite.db .tox/py27/bin/python -m \
    testtools.run neutron.tests.unit...
...
(Pdb) self.engine.url
sqlite:////tmp/iwbgvhbshp.db
```

现在，您可以使用sqlite3检查此文件。

```
$ sqlite3 /tmp/iwbgvhbshp.db
```

### 功能测试

功能测试（neutron/tests/functional/)）旨在验证实际的系统交互。 Mocks 应该谨慎使用，如果有的话。 应该注意确保现有的系统资源不被修改，并且在测试成功和失败的情况下，在测试中创建的资源都得到了适当的清理。 请注意，当在gate运行时，功能测试从源代码编译OVS。 检查*neutron/tests/contrib/gate_hook.sh*。 其他工作目前从软件包中使用OVS。

我们来看看功能测试框架的好处。 Neutron提供了一个名为'ip_lib'的库，其中包含'ip'二进制文件。 其中一种方法称为“device_exists”，它接受设备名称和命名空间，如果设备存在于给定的命名空间中，则返回True。 很容易构建一个直接针对该方法的测试，而这样的测试将被视为“单元”测试。 然而，这样的测试应该使用什么框架呢？ 使用单元测试框架的测试不能在系统上改变状态，因此实际上无法创建设备并声明它现在存在。 这样的测试大致如下：

* 它会mock'执行'，一种执行shell命令的方法来返回一个名为'foo'的IP设备。

* 那么它会assert当'device_exists'且名称为'foo'时，它会返回True，若是其他设备名称则返回False。

* 这很可能会断言'execute'被使用如下命令：`ip link show foo`。

这样的测试的价值是有争议的。记住，新的测试不是免费的，它们需要维护。代码经常被重构，重新实现和优化。

* 还有其他方法可以确定设备是否存在（比如通过查看*/sys/class/net*），在这种情况下，测试将不得不更新。

* 使用他们的名字mock方法。当方法重命名，移动或移除时，必须更新它们的mock。由于可以避免的原因，这样会减缓开发。

* 最重要的是，测试不会断言该方法的行为。它只是断言代码是否如写的那样工作。

当为'device_exists'添加功能测试时，添加了几个框架级方法。 这些方法现在也可以被其他测试使用。 一种这样的方法在命名空间中创建一个虚拟设备，并确保在测试运行结束时清除命名空间和设备，无论使用“addCleanup”方法是成功还是失败。 该测试生成临时设备的详细信息，断言该名称的设备不存在，创建该设备，断言它现在存在，删除它，并断言它不再存在。 如果使用单元测试框架编写，这样的测试可以避免上述所有上述三个问题。

功能测试也用于瞄准较大的范围，如代理。 存在许多很好的例子：请参见OVS，L3和DHCP代理功能测试。 这样的测试针对顶级代理方法，并断言应该执行的系统交互确实被执行。 例如，要测试接受网络属性并配置该网络的dnsmasq的DHCP代理的顶级方法，则测试：

* 实例化一个DHCP代理类的实例（但不启动它的进程）。

* 使用准备的数据调用其顶级功能。

* 创建一个临时命名空间和设备，并从该命名空间调用'dhclient'。

* 声明设备成功获取了预期的IP地址。

### 全栈测试

#### 为什么？

“fullstack”测试背后的想法是填补单元+功能测试和Tempest之间的差距。 Tempest 测试运行成本高昂，并专门针对黑盒API测试。 Tempest需要运行OpenStack部署，这可能难以配置和设置。 根据测试所需的拓扑结构，完全堆栈测试通过照顾部署本身来解决这些问题。 开发人员进一步受益于全栈测试，因为它可以充分模拟真实的环境，并提供快速重现的方式来验证代码，同时还在编写它。

#### 如何做？

全栈测试建立自己的Neutron进程（服务器和代理）。在运行开始之前，他们假设工作的Rabbit和MySQL服务器。有关如何在VM上运行fullstack测试的说明，请参见下面的说明。

每个测试定义了自己的拓扑（什么和多少服务器和代理应该运行）。

由于测试运行在机器本身，全栈测试可以进行“白盒”测试。这意味着您可以通过API创建路由器，然后声明为其创建了命名空间。

全栈测试仅在Neutron资源中运行。 您可以使用Neutron API（中子服务器设置为NOAUTH，以使Keystone不在picture中）。 虚拟机可以使用类似容器的类来模拟：`neutron.tests.fullstack.resources.machine.FakeFullstackMachine`。 其使用示例可以在以下位置找到：`neutron/tests/fullstack/test_connectivity.py`。

全栈测试可以通过启动代理多次来模拟多节点测试。 具体来说，每个节点都有自己的*OVS/inuxBridge/DHCP/L3*代理副本，全部配置有相同的“主机”值。 每个OVS代理连接到它自己的一对*br-int/br-ex*，然后这些桥互连。 对于LinuxBridge代理，每个代理都在自己的命名空间中启动，名为`host- <some_random_value>`。 这些命名空间与OVS `central` 桥连接。

![](https://docs.openstack.org/developer/neutron/_images/fullstack_multinode_simulation.png)

数据库层的分割是通过每个测试创建数据库来保证的。 消息层通过利用称为`vhosts`的RabbitMQ功能来实现分段。 简而言之，就像MySQL服务器提供多个数据库一样，RabbitMQ服务器也可以提供多个消息传递域。 一个`vhost`中的交换和队列与另一个 `vhost` 中的交换和队列分段。

请注意，如果您想使用fullstack测试那些涉及到`python-neutronclient`以及`Neutron`的更改，那么您应该确保您的fullstack测试是一个独立的第三方变化，这取决于`python-neutronclient` 并在更改使用提交信息中的`Depends-On`标签。在您的fullstack测试将在gate工作之前，您将需要等待下一个版本的`python-neutronclient`，`python-neutronclient`的最小版本bump在全局requirements。 这是因为tox使用了`openstack/requirements`存储库中的`upper-constraint.txt`文件中列出的`python-neutronclient`版本。

#### 什么时间？

1. 您想要测试 Neutron 组件（服务器和代理）之间的相互作用，并已通过单元或功能测试隔离测试每个组件。 您应该有许多单元测试，更少的测试来测试组件，甚至更少的测试他们的交互。 边缘情况不应该用全堆栈测试进行测试。

2. 您希望通过测试需要多节点测试的功能（如l2pop，L3 HA和DVR）来增加覆盖范围。

3. 你想测试代理重新启动。 我们在OVS，DHCP和L3代理中发现了错误，并没有找到测试这些场景的有效方式。 完整的堆栈测试可以在这里起到作用，因为完整的堆栈基础设施可以在测试期间重新启动代理。

#### 实例

Neutron提供了服务质量API，最初在端口级提供带宽上限。 在参考实现中，它通过使用OVS功能来实现。
 `neutron.tests.fullstack.test_qos.TestQoSWithOvsAgent.test_qos_policy_rule_lifecycle`是一个积极的例子，说明如何使用fullstack测试基础架构。 它创建一个网络，子网，QoS策略和规则以及使用该策略的端口。 然后断定在连接到该端口的OVS桥上存在期望的带宽限制。 该测试是一个真正的集成测试，它的意义在于它调用API，然后断言Neutron适当地与管理程序进行交互。

### API 测试

API测试（neutron/tests/tempest/api/）旨在确保Neutron API的功能和稳定性。 尽可能地，对路径的更改不应该与代码的更改同时进行，以限制引入向后兼容的更改的可能性，尽管引入新API的同一补丁应包括API测试。

由于API测试针对未经测试管理的部署Neutron守护程序，因此不应依赖于控制目标守护程序的运行时配置。 API测试应该是黑盒 - 不应该对实现做出假设。 应该验证Neutron的REST API定义的合同，并且所有与守护进程的交互应通过REST客户端进行。

Neutron的 */tests/tempest/api* 目录从Kilo框架的Tempest项目中复制出来。 当时，Tempest和Neutron存储库之间的测试重叠。 然后通过雕刻出属于`Tempest`的资源子集，在Neutron中消除了这种重叠。

属于Tempest的API测试涉及Neutron资源的一个子集：

* Port
* Network
* Subnet
* Security Group
* Router
* Floating IP

这些资源导出被使用。它们在大多数Neutron部署中被发现，无论插件如何，并且直接参与实例的网络和安全性。 它们是Neutron的最低需求。

这在大多数情况下不包括这些资源的扩展（例如：对子网的额外DHCP选项，或snat_gateway模式到路由器）。

应向Neutron存储库提供其他资源的测试。 场景测试应该根据他们所针对的API在Tempest和Neutron之间分类。

### 情景测试

场景测试（中子/测试/暴风/场景），类似于API测试，使用Tempest测试基础设施并具有相同的要求。 在Tempest开发人员指南中可以找到编写良好场景测试的指南：http://docs.openstack.org/developer/tempest/field_guide/scenario.html

根据测试针对的Neutron API，场景测试（如API测试）分为Tempest和Neutron存储库。

### Rally Tests

Rally 测试（rally-jobs/plugins）使用rally 基础设施来执行Neutron部署。 在rally 插件文档中可以找到编写好的rally 测试的准则。这里还有一些例子，将rally 插件添加到中子的过程需要三个步骤：1）编写一个插件并将其放在*rally-jobs/plugins/*下。 这是你的rally场景 2）（可选）在*rally-jobs/extra/*下添加安装文件。 这是确保您的环境可以成功处理您的方案请求所需的任何devstack配置; 3）编辑neutron-neutron.yaml。 这是您的方案“合同”或SLA。

## 开发过程

预计为合并而提出的任何新的更改都将附带该功能或代码区域的测试。 所提交的任何错误修复也必须有测试证明他们被修复！ 此外，在提出合并之前，所有当前的测试都应该过去。

### 单元测试的架构

单元测试树的结构应该与代码树的结构匹配，例如

```
- target module: neutron.agent.utils

- test module: neutron.tests.unit.agent.test_utils
```

单位测试模块应在*neutron/tests/unit/*下具有与目标模块在*neutron/*下相同的路径，其名称应为以test_为前缀的目标模块的名称。 这个要求旨在使开发人员更容易找到给定模块的单元测试。

类似地，当测试模块定位到一个软件包时，该模块的名称应该是以test_为前缀的软件包的名称，其路径与测试目标模块相同。

```
- target package: neutron.ipam

- test module: neutron.tests.unit.test_ipam
```

可以使用以下命令验证单元测试树是否按照上述要求进行结构化：

```
./tools/check_unit_test_structure.sh
```

在适当的情况下，可以为上述脚本添加例外。 例如，如果代码不是Neutron命名空间的一部分，则可以将其单元测试从检查中排除是合理的。

*注意：生产代码在任何时候都不会从测试子树（neutron.tests）导入任何东西。 有一些发行版在一个单独的软件包中分离出中子模块，这个软件包默认情况下不会安装，使任何依赖模块存在的代码都会失败。 例如，RDO是其中之一。*

## 运行测试

在提交补丁之前，您应该始终确保所有测试通过; 一个tox run是由genit执行的jenkins gate引发的每个补丁推送审查。

像其他OpenStack项目一样，中子使用tox来管理运行测试用例的虚拟环境。 它使用Testr来管理测试用例的运行。

Tox处理创建一系列针对特定版本的Python的virtualenvs。

Testr处理并行执行一系列测试用例以及跟踪长时间运行的测试和其他事情。

有关OpenStack使用的基于标准Tox的测试基础架构的更多信息以及如何使用Testr进行一些常见的测试/调试过程，请参阅此wiki页面：https://wiki.openstack.org/wiki/Testr

### PEP8 和单元测试

运行pep8和单元测试与在Neutron源代码的根目录中执行此操作一样简单：

```
tox
```

只运行 pep8

```
tox -e pep8
```

由于pep8包括在所有文件上运行pylint，所以运行可能需要相当长的时间。 将pylint检查限制为只有最新修补程序更改所更改的文件：

```
tox -e pep8 HEAD~1
```

只运行单元测试

```
tox -e py27
```

### 功能测试

要运行不需要sudo权限或特定系统依赖关系的功能测试：

```
tox -e functional
```

要运行所有功能测试，包括需要sudo权限和特定于系统的依赖关系的功能测试，应该遵循由*tools/configure_for_func_testing.sh*定义的过程。

重要信息：*configure_for_func_testing.sh*依赖于DevStack对底层主机进行大量修改。 脚本的执行需要sudo权限，建议仅在干净且可拆卸的虚拟机上调用以下命令。 已经安装了DevStack的VM也是可以的。

```
git clone https://git.openstack.org/openstack-dev/devstack ../devstack
./tools/configure_for_func_testing.sh ../devstack -i
tox -e dsvm-functional
```

'-i'选项是可选的，并指示脚本使用DevStack来安装和配置所有Neutron的软件包依赖项。 如果DevStack已经用于将Neutron部署到目标主机，则不需要提供此选项。

### 全栈测试

要运行全栈测试，你需要运行：

```
tox -e dsvm-fullstack
```

由于全栈测试通常需要与功能测试相同的资源和依赖关系，因此建议使用配置脚本*tools/configure_for_func_testing.sh*（如上所述）。 首次在干净的VM上运行全栈测试时，我们建议成功运行./stack.sh，以确保满足所有Neutron的依赖关系。 基于全栈的Neutron守护程序会将日志生成到*/opt/stack/logs/dsvm-fullstack-logs*中的子文件夹中（例如，名为“test_example”的测试将生成日志到*/opt/stack/logs/dsvm-fullstack-logs/test_example/*），所以如果你的测试失败，这将是一个很好的地方。 从测试基础架构本身登录放在：*/opt/stack/logs/dsvm-fullstack-logs/test_example.log*中。 Fullstack测试套件假设测试机的根命名空间中的240.0.0.0/4（E类）范围可用于其使用。

### API以及情景测试

要运行api或场景测试，请使用DevStack部署Tempest和Neutron，然后从tempest目录运行以下命令：

```
tox -e all-plugin
```

如果要限制要运行的测试数量，可以执行以下操作：

```
export DEVSTACK_GATE_TEMPEST_REGEX="<you-regex>" # e.g. "neutron"
tox -e all-plugin $DEVSTACK_GATE_TEMPEST_REGEX
```

### 运行单独的测试

对于运行单独的测试模块，案例或测试，您只需要将所需的点分隔路径作为参数传递给它。

例如，以下只运行一个测试或测试用例：

```
$ tox -e py27 neutron.tests.unit.test_manager
$ tox -e py27 neutron.tests.unit.test_manager.NeutronManagerTestCase
$ tox -e py27 neutron.tests.unit.test_manager.NeutronManagerTestCase.test_service_plugin_is_loaded
```

如果要将其他参数传递给ostestr，可以执行以下操作:

```
$ tox -e -epy27 – –regex neutron.tests.unit.test_manager –serial
```

## 测试覆盖范围

中子具有快速增长的代码库，并且有很多领域需要更好的覆盖。

要掌握需要测试的领域，您可以通过运行以下步骤来检查当前的单元测试覆盖范围：

```
$ tox -ecover
```

由于coverage命令只能显示单元测试覆盖范围，因此将保留覆盖文档，该文档显示以下文档中的代码区域的测试覆盖：*doc/source/devref/testing_coverage.rst*。 您也可以依赖于Zuul日志，这些日志是合并后生成的（不是每个项目都构建覆盖结果）。 要访问它们，请执行以下操作：

* 查看最新的[合并提交](https://review.openstack.org/gitweb?p=openstack/neutron.git;a=search;s=Jenkins;st=author)

* 转到：http://logs.openstack.org/<first-2-digits-of-sha1>/<sha1>/post/neutron-coverage/。

* spec 是一项正在进行的工作，以提供更好的landing page。

## 调试

默认情况下，运行测试时，对pdb.set_trace（）的调用将被忽略。 要使pdb语句工作，请按如下所示调用tox：

```
$ tox -e venv -- python -m testtools.run [test module path]
```

Tox创建的虚拟环境（venv）也可以在tox运行后被激活并重新用于调试：

```
$ tox -e venv
$ . .tox/venv/bin/activate
$ python -m testtools.run [test module path]
```

Tox软件包并在每次调用时将Neutron源代码树安装在给定的静态文件中，但是如果需要在调用之间进行修改（例如，添加更多pdb语句），则建议将源代码树以可编辑模式安装到venv中：

```
# run this only after activating the venv
$ pip install --editable .
```

可编辑模式可确保对源代码树进行的更改会自动反映在venv中，并且这些更改在下一次运行时不会被覆盖。

### Post-mortem Debugging

TBD: how to do this with tox.

### References

[1]	PUDB debugger: https://pypi.python.org/pypi/pudb