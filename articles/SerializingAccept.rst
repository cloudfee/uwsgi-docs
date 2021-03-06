序列化accept(), 亦称惊群效应，亦亦称Zeeg难题
===============================================================

UNIX世界的一个历史问题是“惊群效应（thundering herd）”。

它是什么呢？

比方说，一个进程绑定到一个网络地址 (它可能是 ``AF_INET``,
``AF_UNIX`` 或者任意你想要的) ，然后fork自己：

.. code-block:: c

   int s = socket(...)
   bind(s, ...)
   listen(s, ...)
   fork()

在多次fork自己之后，每个进程一般将会开始阻塞在 ``accept()`` 上

.. code-block:: c

   for(;;) {
       int client = accept(...);
       if (client < 0) continue;
       ...
   }

这个有趣的问题是，在较老的／古典的UNIX上，每当socket上尝试进行一个连接，阻塞在 ``accept()`` 上的每个进程的 ``accept()`` 都会被唤醒。

只有其中一个进程能够真正接收到这个连接，而剩余的进程将会获得一个无聊的 ``EAGAIN`` 。

这导致了大量的CPU周期浪费 (内核调度器必须把控制权交给所有等待那个socket的休眠中的进程)。

当你使用线程来代替进程的时候，这种行为 (出于各种原因) 会被放大 (因此，你有多个阻塞在 ``accept()`` 的线程)。

实际解决方法是把一个锁放在 ``accept()`` 调用之前，来序列化它的使用：

.. code-block:: c

   for(;;) {
       lock();
       int client = accept(...);
       unlock();
       if (client < 0) continue;
       ...
   }

对于线程来说，处理锁一般更容易，但是对于进程，你必须利用系统特定的解决方案，或者回退到历史悠久的SysV ipc
子系统 (稍后会有更多关于这个的信息)。

到了近代，绝大多数的UNIX系统都已经进化了，现在，内核（或多或少）确保在一个连接事件上，只有一个进程/线程会被唤醒。

好啦，问题已解决，那我们在谈论什么呢？

select()/poll()/kqueue()/epoll()/...
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在前1.0时代，uWSGI比目前的形式简单得多 (并且更无趣)。它并没有信号框架，并且不能够监听多个地址；为此，它的循环引擎只在每个进程／线程中调用 
``accept()`` ，因此惊群效应 (多亏了现代内核) 并不是一个问题。

进化是有代价的，因此不久，uWSGI进程／线程的标准循环引擎从

.. code-block:: c

   for(;;) {
       int client = accept(s, ...);
       if (client < 0) continue;
       ...
   }

移到了一个更复杂的：

.. code-block:: c

   for(;;) {
       int interesting_fd = wait_for_fds();
       if (fd_need_accept(interesting_fd)) {
           int client = accept(interesting_fd, ...);
           if (client < 0) continue;
       }
       else if (fd_is_a_signal(interesting_fd)) {
           manage_uwsgi_signal(interesting_fd);
       }
       ...
   }

问题现在是 ``wait_for_fds()`` 样例函数：它会调用某些例如 ``select()``, ``poll()`` 或者更现代的 ``epoll()`` 和
``kqueue()`` 方法。

这些类型的系统调用是文件描述符的“监控器”，并且它们会在所有等待同一个文件描述符的进程／线程中被唤醒。

在你开始指责你的内核开发者之前，你应该知道，这是正确的方法，因为内核并不能知道你是在等待那些文件描述符来调用
``accept()`` ，还是做些更有趣的事。

所以，欢迎再次回到惊群效应。

应用服务器 VS web服务器
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

流行的、久经检验的稳定的多进程参考web服务器是Apache
HTTPD.

它在IT演化的几十年中生存了下来，并且仍然是驱动整个互联网的最重要的技术之一。

天生仅多进程的特性，Apache必须始终处理惊群效应问题，并且它们用SysV ipc信号量来解决它。

(注：对于此，Apache真的很智能，当它仅需等待单个文件描述符时，它只调用 ``accept()`` ，利用现代内核的反惊群效应策略的优势)

(更新：Apache 2.x甚至允许你选择使用哪个锁技术，包括用于非常古老的系统的flock/fcntl，但是在绝大多数的系统上，当处于多进程模式的时候，它会使用sysv信号里)

甚至在当代的Apache版本上，strace它的一个进程 (绑定到多个接口)，你会看到像这样的东东 (它是一个Linux系统):

.. code-block:: c

   semop(...); // lock
   epoll_wait(...);
   accept(...);
   semop(...); // unlock
   ... // manage the request

SysV信号量保护你的epoll_wait免受惊群效应之扰。

所以，另一个问题解决了，这个世界真是一个辣么美好的地方……，但是……

**SysV IPC对于应用服务器并不好 :(***

“应用服务器”的定义是非常通用的，在这种情况下，我们指的是，一个或多个有非特性（非root）用户生成的进程，绑定到一个或多个网络地址上，允许自定义的、高度不确定的代码。

即使你对SysV IPC是如何工作的有最小／基本的了解，你都会知道它的每个部件都是系统中的受限资源 (而在现代的BSD中，这些限制被设置成低的离谱的值，PostgreSQL FreeBSD用户对这个问题深有感触)。

仅需在终端允许'ipcs'，来获取你的内核中的分配对象的列表。是哒，在你的内核中。SysV ipc对象是持久化资源，它们需要由用户手动移除。与那些可以分配数以百计的那样的对象，并且填充你的受限SysV IPC内存的相同的用户。

Apache世界中由SysV ipc使用引发的最常见的问题之一是当你粗暴地杀死Apache实例时引发的泄漏 (是哒，你永远不应该这样做，但是如果勇敢/傻得在你的web服务器进程中托管不可靠的PHP应用的话，那么你别无选择)。

要更好地理解它，请生成Apache，然后 ``killall -9 apache2`` 。重新生成它，然后运行'ipcs'，你将每次都会获得一个新的信号量对象。你知道问题所在了吧？ (给Apache大师：是哒，我知道有hack技巧来避免，但是这是默认的行为)

Apache一般是一个系统服务，由有意识的系统管理员管理，因此，除少数情况外，你可以继续信任它几十年，即使是在它决定使用更多的SysV ipc对象的情况下 :)

可悲的是，你的应用服务器是由不同类型的用户管理的，从最熟练的到那种应该立即换工作的，到那种站点被想要控制你的服务器的白痴破解的。

应用服务器并不危险，用户才是。而应用服务器是由用户运行的。这个世界就是如此的丑陋。

应用服务器开发者是如何解决它的
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

快速回答：他们一般不解决／在乎它

注意：我们谈论的是多进程，我们已经看到多线程是很容易解决的。

提供静态文件或者代理 (一个web服务器的主要活动) 一般是一种快速非阻塞 (在各个观点下非常确定) 的活动。相反，web应用则是更慢更重的方式，因此，即使在中等负载的网站上，休眠进程的数量一般都是低的。

在高负载站点上，你会祈祷空闲进程，而在无负载的站点上，惊群效应问题是完全不相干的（除非你在386上运行你的站点）。

鉴于你通常分配给应用服务器相对较少数量的进程，我们可以说惊群效应不是个问题。

另一个方法是动态进程生成。如果你确定你的应用服务器总是运行最少必须数量的进程，那么你会高度减少惊群效应问题。 (看看--cheaper uWSGI选项家族)

没问题？？？所以，再次，我们在谈论什么？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我们在谈论的是“常见情况”，而对于常见情况，有过多有效选择 (明显，不是uWSGI) ，而我们在谈论的大多数问题是不存在的。

由于uWSGI项目的开始，是🈶️托管公司开发的，其中，“常见情况”并不存在，因此我们很多关注的是极端情况问题，奇怪的设置，以及那些绝大多数的用户绝不需要关心的问题。

除此之外，uWSGI支持只在一般用途的web服务器，例如Apache（我不得不说，Apache可能是唯一一个一般用途的web服务器，因为它以一种相对安全和稳定的方式，允许在其进程空间中基本任何操作），才常见的／可用的操作模式 ，因此大量的与用户不良行为合并的新问题出现了。

uWSGI最具挑战的开发阶段之一是添加多线程。线程是强大的，但是真多很难以正确的方式进行管理。

线程是种比进程便宜的方式，因此，一般来说，你会为你的应用分配数十个线程 (记住，未使用内存就是浪费的内存)。

几十 (或几百) 个等待同组文件描述符的线程把我们带回了惊群效应问题 (除非你所有的线程都在不断地被使用)。

出于这样的理由，当你在uWSGI中启用多线程时，会分配一个pthread互斥锁，在每个线程中序列化epoll()/kqueue()/poll()/select()...使用。

另一个问题解决啦 (并且对于uWSGI来说很奇怪，并不需要使用选项 ;)

但是……

Zeeg难题：带多线程的多进程
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在2013年6月27日，David Cramer写了一篇有趣的博文 (你可能不同意它的结论，但是现在这没关系，你可以继续安全地痛恨
uWSGI，或者嘲笑其命名选择，又或者是它选项的数目)。

http://cramer.io/2013/06/27/serving-python-web-applications

David面临的问题是，惊群效应如此强大，以致于它的响应时间被它破坏了 (非恒定性能是其测试的主要结果)。

为什么它会发生呢？uWSGI分配的互斥锁不是已经解决了这个问题了吗？

David运行的uWSGI有10个进程，每个进程有10个线程：

.. code-block:: sh

   uwsgi --processes 10 --threads 10 ...

虽然互斥锁保护单一进程中的每个线程在同个请求上调用 ``accept()`` ，但是并没有这样的机制（或者更好的，并不是默认启用它的，见下）保护多进程不受其害，因此给定可用于管理请求的线程数（100），单个进程不可能完全阻塞 (说明：它所有的10歌线程都阻塞在一个请求中)，所以，欢迎回到惊群效应。

David是如何解决它的？
^^^^^^^^^^^^^^^^^^^^^

uWSGI是一个有争议的软件，这并不可耻。有些用户极度痛恨它，而有些则病态地热爱它，但是所有人都同意文档可能更好 ([OT] 当所有的人都同意某事的时候是不错的，但是uwsgi-docs上的pull请求数低得囧囧有神，并且所有都来自相同的人……来吧，帮帮我们！！！)

David使用了一个经验方法，发现它的问题，然后决定通过运行绑定在不同socket上的独立uwsgi进程，并配置nginx在它们之间轮询来解决它。

这是一个非常优雅的方法，但它有一个问题：nginx不能知道发送请求的进程是否所有的线程都处在忙碌状态。这有用，但是是一个次优解。

最好的方法是有一个内部进程锁住 (就像Apache)，同时在线程和进程中序列化所有的 ``accept()`` 

uWSGI文档太糟糕了： --thunder-lock
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Michael Hood (你也会在David博文的评论中发现他的名字)在前段时间于uWSGI的邮件列表／问题跟踪器中标记了这个问题，他甚至出了一个以 ``--thunder-lock`` 选项为结果的初始补丁 (这就是为什么开源更好 ;)

``--thunder-lock`` 自uWSGI 1.4.6起可用，但从未在文档中记录过 (任何形式)

只有那些关注邮件列表 (或者面对具体问题) 的人才知道它。

SysV IPC信号量不好，你如何解决它？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

进程间锁自uWSGI 0.0.0.0.0.1起就是个问题了，但是我们在项目的第一个公开版本 (in 2009) 中解决了它。

我们基本上检查每个操作系统功能，然后选择它们提供的最好／最快的ipc锁，给我们的代码填充了数十个
#ifdef。

当你启动uWSGI时，你应该可以在它的日志中看到已选择了哪个“锁引擎”。

支持它们中的许多种：

 - _PROCESS_SHARED 和 _ROBUST属性的pthread互斥锁 (现代Linux和Solaris)
 - 带_PROCESS_SHARED的pthread互斥锁 (较老的Linux)
 - OSX自旋锁 (MacOSX, Darwin)
 - Posix信号量 (FreeBSD >= 9)
 - Windows互斥锁 (Windows/Cygwin)
 - SysV IPC信号量 (对所有其他系统回退)

对于uWSGI特有的特性，例如缓存、rpc，都需要它们的使用，而所有那些特性都要求改变共享内存结构 (通过
mmap() + _SHARED分配)

每个引擎彼此间不同，而处理它们已经很痛苦，而（更重要的是）它们有些并不“健壮”。

"健壮"一词是从phread借来的。如果一个锁是”健壮“的，那么意味着如果锁住的进程死掉，那么会释放该锁。

你可能会认为所有的锁引擎都有这个特点，但遗憾的是，只有少数几个可靠工作。

出于这个原因，uWSGI master进程必须分配一个额外的线程
(“死锁”检测器) 来不断地检测映射到死掉的进程的非健壮的未释放锁。

这很痛苦，但是，任何告诉你IPC锁容易的人都应该加入JEDI学校……

uWSGI开发者特么是懦夫
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

David Cramer和Graham Dumpleton (是哒，他是mod_wsgi的作者，但是在很大程度上促成了uWSGI以及其他WSGI服务器的发展，这就是为什么开源更好的另一个理由) 都问为什么当请求多进程+多线程时， ``--thunder-lock`` 并不是默认项。

这是一个不错的问题，它有一个简单的回答：我们是只在乎钱的懦夫。

uWSGI完全开源，但是它的开发是由使用它的公司和Unbit.it客户赞助的（以多种方式）。

为一个“常见的”使用（例如多进程+多线程）启用“高风险”特性对我们而言太难了，除此之外，库／内核不兼容的情况（特别是在Linux上）真多很痛苦。

例如，要拥有健壮的pthread互斥锁，你需要一个带有现代glibc的现代内核，但是，常用的发行版 (像centos家族) 混合了较老的内核，具有较新的glibc以及较老的glibc。这导致了不能正确检测对于一个平台而言，哪个是最好的锁引擎，因此，当uwsgiconfig.py脚本不能肯定时，它退到最安全的方式 (例如Linux上的非健壮pthread互斥锁)。

死锁检测器应该让你免于大部分的问题，但是“应该”这一词是关键。在这种代码上编写测试套件（或者甚至是单个单元测试）基本上不可能 (呃，至少对我而言)，所以我们不能确保所有都是正确的 (而报告线程错误对于用户，以及熟练的开发者而言都很难，除非你工作在pypy上 ;)

Linux pthread健壮的互斥锁是可靠的，我们“相当”肯定，因此你应该能够在现代Linux系统上99.999999%成功启用 ``--thunder-lock`` ，但是，我们更愿意（目前）让用户自觉启用它。

当SysV IPC信号量是一个更好的选择时
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

是哒，有些情况下，SysV IPC信号量比系统特定的特性带给你更好的结果。

Booking.com的Marcin Deranek已经考验uWSGI多月了，并且帮助我们修复极端情况（甚至在锁领域）。

他指出，系统特定的锁引擎倾向于内核调度 (当unlock之后选择哪个进程赢得下一个错的时候) ，而不是轮询分布。

对于他们那种在进程之间等量分布请求会更好的特定需求 (他们通过perl使用uWSGI，因此没有任何线程，但是会生成大量的进程)，他们（当前）选择使用"ipcsem" 锁引擎：

.. code-block:: sh

   uwsgi --lock-engine ipcsem --thunder-lock --processes 100 --psgi ....

（这个时候）有趣的是，你可以很容易测试锁是否运行良好。仅需开始压测服务器，你将会看到请求日志中报告pid是怎样每次不同的，而使用系统特定的锁，pid是相当随机的，并且带有喜欢最后使用的进程的严重倾向。

够有趣的是，他们面对的第一个问题是ipcsem泄漏（当你处在紧急情况下，优雅重载／停止就是你的敌人，而kill -9将是你的银弹）

要解决这个问题，可以用--ftok选项，允许你分配一个唯一的id给信号量对象，并且在它从前一个允许中可用的情况下重用它：

.. code-block:: sh

   uwsgi --lock-engine ipcsem --thunder-lock --processes 100 --ftok /tmp/foobar --psgi ....

--ftok接收一个文件作为参赛，它将使用它来构建唯一的id。常见的模式是为它使用pidfile


其他可移植的锁引擎又如何？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

除了"ipcsem"外，uWSGI 也(在可用之处) 添加了"posixsem"。

只在FreeBSD >= 9时默认使用它们，但在Linux上也可用。

它们并不“健壮”，但是它们并不需要内核资源，因此如果你相信我们的死锁检测器，那么它们就是一个相当棒的方法。 (注意：Graham
Dumpleton 像我指出一个事实，它们也可以在Apache 2.x上启用)

总结
^^^^^^^^^^^

你可以拥有全宇宙最好（或者最坏）的软件，但是没有文档，就不可能。

Apache团队仍然碾压我们绝大多数想要伸向他们市场份额的人 :)

福利部分：以uWSGI友好的方式使用Zeeg方法
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我必须承认，我并不是supervisord的忠实粉丝。毫无疑问，它是一个好软件，但我认为Emperor和--attach-daemon设施是部署问题的一个更好的方法。除此之外，如果你想要一个“脚本化”/“可扩展”的进程监管，那么我认为Circus
(https://circus.readthedocs.io/) 更加有趣，更加可行 (我在uWSGI Emperor中实现了socket激活之后做的第一件事就是为Circus中的相同特性进行pull请求 [已合并，如果你关心的话])。

显然，supervisord能用，并且很多人使用它，但是，作为一个重度uWSGI用户，我倾向于尽可能地使用它的特性来完成。

我可以用的第一个方法是绑定到10个不同的端口，然后将其每个映射到一个指定的进程：

.. code-block:: ini

    [uwsgi]
    processes = 5
    threads = 5

    ; create 5 sockets
    socket = :9091
    socket = :9092
    socket = :9093
    socket = :9094
    socket = :9095

    ; map each socket (zero-indexed) to the specific worker
    map-socket = 0:1
    map-socket = 1:2
    map-socket = 2:3
    map-socket = 3:4
    map-socket = 4:5

现在，你有了一个监控5个进程的master，每个进程绑定到一个不同的地址 (无需 ``--thunder-lock`` )

对于Emperor粉丝，你可以做一个这样的模板 (称之为foo.template):

.. code-block:: ini

    [uwsgi]
    processes = 1
    threads = 10
    socket = :%n

现在，对每个你想要生成的实例+端口进行符号链接：

.. code-block:: sh

    ln -s foo.template 9091.ini
    ln -s foo.template 9092.ini
    ln -s foo.template 9093.ini
    ln -s foo.template 9094.ini
    ln -s foo.template 9095.ini
    ln -s foo.template 9096.ini

福利部分2: 安全的SysV IPC semaphores
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

我司的托管平台重度基于Linux cgroups和名字空间。

第一个 (cgroups) 被用于限制/负责资源使用，而第二个（名字空间）被用于向用户提供一个“分离”系统视图 (例如看到专用的主机名或者根文件系统)。

因为我们允许用户在他们的账号中生成PostgreSQL实例，因此我们需要限制SysV对象。

幸运的是，现代的Linux内核有一个用于IPC的名字空间，因此调用
unshare(CLONE_NEWIPC) 将会创建一个完整的新IPC对象集合 (与其他分离)。

在客户专用的Emperor中调用 ``--unshare ipc`` 是一个常见的方法。当与内存cgroup结合在一起的时候，你将得到一个相当安全的设置。


关于作者:
^^^^^^^^

作者： Roberto De Ioris

修正人： Honza Pokorny
