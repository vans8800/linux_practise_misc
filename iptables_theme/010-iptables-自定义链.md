## 背景信息
---

iptables的默认链就已经能够满足需求，为什么还需要自定义链呢？

原因如下：当默认链中的规则非常多时，不方便管理。

> 想象一下，如果INPUT链中存放了200条规则，这200条规则有针对httpd服务的，有针对sshd服务的，有针对私网IP的，有针对公网IP的
>
> 假如，我们突然想要修改针对httpd服务的相关规则，难道我们还要从头看一遍这200条规则，找出哪些规则是针对httpd的吗？这显然不合理。

所以，iptables中，可以自定义链，通过自定义链即可解决上述问题。

## 自定义链
---

> 假设，要自定义一条链，链名叫IN_WEB，可以将所有针对80端口的入站规则都写入到这条自定义链中，当以后想要修改针对web服务的入站规则时，就直接修改IN_WEB链中的规则即可。
>
> 即使默认链中有再多的规则，所有针对80端口的入站规则都存放在IN_WEB链中，
>
> 同理，将针对sshd的出站规则放入到OUT_SSH自定义链中，将针对Nginx的入站规则放入到IN_NGINX自定义链中。

但自定义链并不能直接使用，而是需要被默认链引用才能够使用。

### 创建自定义链 

使用-N选项可以创建自定义链，示例如下

<img width="756" height="290" alt="image" src="https://github.com/user-attachments/assets/b5603b66-f915-4448-b597-a7338271ef5e" />


”-t filter”表示操作的表为filter表，与之前的示例相同，省略-t选项时，缺省操作的就是filter表。

“-N IN_WEB”表示创建一个自定义链，自定义链的名称为”IN_WEB”

自定义链创建完成后，查看filter表中的链，这条自定义链的引用计数为0 (0 references)，也即，这条自定义链还没有被任何默认链所引用。

所以，即使IN_WEB中配置了规则，也不会生效。

如下图所示，配置一些规则用于举例。

<img width="729" height="176" alt="image" src="https://github.com/user-attachments/assets/d1a9f82e-782b-4236-baa6-75abfc86872d" />

如上图，对自定义链的操作与对默认链的操作并没有什么不同，一切按照操作默认链的方法操作自定义链即可。

### 引用和修改自定义链

既然IN_WEB链是为了针对web服务的入站规则而创建的，那么这些规则应该去匹配入站的报文，所以，应该用INPUT链去引用它。

当然，自定义链在哪里创建，应该被哪条默认链引用，取决于实际的工作场景，因为此处示例的规则是匹配入站报文，所以在INPUT链中引用自定义链。

示例如下。

<img width="912" height="309" alt="image" src="https://github.com/user-attachments/assets/7be81e43-4db4-4e19-b937-d1ff02b455f0" />

上图中，在INPUT链中添加了一条规则，访问本机80端口的tcp报文将会被这条规则匹配到

而上述规则中的”-j IN_WEB”表示：访问80端口的tcp报文将由自定义链”IN_WEB”中的规则进行处理。

没错，在之前的示例中，使用”-j”选项指定动作，而此处，将”动作”替换为了”自定义链”，当”-j”对应的值为一个自定义链时，表示被当前规则匹配到的报文将交由对应的自定义链处理，具体怎样处理，取决于自定义链中的规则。

当IN_WEB自定义链被INPUT链引用以后，发现IN_WEB链的引用计数已经变为1，表示这条自定义链已经被引用了1次，自定义链还可以引用其他的自定义链。

”动作”在iptables中被称为”target”，这样描述并不准确，因为target为目标之意，报文被规则匹配到以后，target能是一个”动作”，target也能是一个”自定义链”。

当target为一个动作时，表示报文按照指定的动作处理，当target为自定义链时，表示报文由自定义链中的规则处理。

此刻，在192.168.1.139上尝试访问本机的80端口，已经被拒绝访问，证明刚才自定义链中的规则已经生效。

<img width="380" height="75" alt="image" src="https://github.com/user-attachments/assets/fe716afc-4384-45ed-976b-ea2f78592be2" />

<img width="435" height="347" alt="image" src="https://github.com/user-attachments/assets/fc27eb2c-7ec9-4ee9-9ae1-b8b5c9952beb" />

如上图，使用”-E”选项可以修改自定义链名，引用自定义链处的名称会自动发生改变。

### 删除自定义链

使用”-X”选项可以删除自定义链，但是删除自定义链时，需要满足两个条件：

1、自定义链没有被任何默认链引用，即自定义链的引用计数为0。

2、自定义链中没有任何规则，即自定义链为空。

那么，来删除自定义链WEB试试。


<img width="354" height="39" alt="image" src="https://github.com/user-attachments/assets/41364b22-eff8-458e-92cf-3dde3ec6ee43" />

如上图，使用”-X”选项删除对应的自定义链，但是上例中，并没有成功删除自定义链WEB，提示：Too many links，是因为WEB链已经被默认链所引用，不满足上述条件1。

所以，需要删除对应的引用规则，示例如下。

<img width="441" height="139" alt="image" src="https://github.com/user-attachments/assets/8c0e7587-3258-48f4-ad53-d04f3f737df1" />

如上图，删除引用自定义链的规则后，再次尝试删除自定义链，提示：Directory not empty。 这是因为WEB链中存在规则，不满足上述条件2，所以，需要清先空对应的自定义链:

<img width="531" height="244" alt="image" src="https://github.com/user-attachments/assets/acc59e98-0dea-49d6-a22d-685286ce6f72" />


如上图，使用”-X”选项可以删除一个引用计数为0的、空的自定义链。

## 小结

### 创建自定义链

```bash
#示例：在filter表中创建IN_WEB自定义链
iptables -t filter -N IN_WEB
 ```

### 引用自定义链

```bash
#示例：在INPUT链中引用刚才创建的自定义链
iptables -t filter -I INPUT -p tcp --dport 80 -j IN_WEB
``` 

### 重命名自定义链

```bash
#示例：将IN_WEB自定义链重命名为WEB
iptables -E IN_WEB WEB
 ```

### 删除自定义链

删除自定义链需要满足两个条件

1、自定义链没有被引用

2、自定义链中没有任何规则

```bash
#示例：删除引用计数为0并且不包含任何规则的WEB链
iptables -X WEB
```
