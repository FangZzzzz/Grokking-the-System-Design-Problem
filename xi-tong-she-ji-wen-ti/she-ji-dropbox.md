# 设计 Dropbox

## 设计 Dropbox

让我们设计一个文件托管服务，比如 Dropbox 或 Google Drive。云文件存储使用户能够将他们的数据存储在远程服务器上。通常，这些服务器由云存储提供商维护，并通过网络（通常通过 Internet）提供给用户。用户按月为其云数据存储付费。

类似服务：OneDrive、Google Drive

难度级别：中

### 1、 为什么选择云存储？

云文件存储服务最近变得非常流行，因为它们简化了多个设备之间数字资源的存储和交换。从使用单个个人电脑到使用具有不同平台和操作系统的多个设备（例如智能手机和平板电脑）的转变，每个设备都可以随时从不同的地理位置进行便携式访问，这被认为是云存储服务大受欢迎的原因。

以下是此类服务的一些主要好处：

**可用性：**云存储服务的座右铭是随时随地提供数据。用户可以随时随地从任何设备访问他们的文件/照片。

**可靠性和持久性：**云存储的另一个好处是它提供了 100% 的数据可靠性和耐用性。云存储通过将数据的多个副本存储在不同地理位置的服务器上，确保用户永远不会丢失他们的数据。

**可扩展性：**用户永远不必担心存储空间不足。有了云存储，只要您准备好付费，您就可以拥有无限的存储空间。

如果您以前没有使用过[dropbox.com](http://dropbox.com/)，我们强烈建议您在那里创建一个帐户并上传/编辑文件，并通过他们的服务提供的不同选项。这对你理解本章有很大帮助。

### 2、系统的要求和目标

> 💡 你应该总是在面试开始时明确要求。请务必提出问题，以找到面试官所考虑的系统的确切范围。

我们希望从云存储系统中获得什么？以下是我们系统的顶级要求：

1. 用户应该能够从任何设备上传和下载他们的文件/照片。
2. 用户应该能够与其他用户共享文件或文件夹。
3. 我们的服务应该支持设备之间的自动同步，即在一个设备上更新文件后，它应该在所有设备上同步。
4. 系统应支持存储高达 GB 的大文件。
5. ACID是必需的。应保证所有文件操作的原子性、一致性、隔离性和持久性。
6. 我们的系统应该支持离线编辑。用户应该能够在离线时添加/删除/修改文件，并且一旦他们上线，他们的所有更改都应该同步到远程服务器和其他在线设备。

**扩展要求**

* 系统应该支持数据的快照，以便用户可以返回到文件的任何版本。

### 3、一些设计考虑

* 我们应该预计巨大的读写量。
* 预计读写比率几乎相同。
* 在内部，文件可以存储在小部分或块中（例如 4MB）；这可以提供很多好处，即所有失败的操作只能针对文件的较小部分重试。如果用户上传文件失败，那么只会重试失败的块。
* 我们可以通过仅传输更新的块来减少数据交换量。
* 通过删除重复的块，我们可以节省存储空间和带宽使用。
* 客户端保存元数据（文件名、大小等）的本地副本可以为我们节省大量往返服务器的时间。
* 对于小的更改，客户端可以智能地上传差异而不是整个块。

### 4、容量估计和约束

* 假设我们有 5 亿总用户和 1 亿日活跃用户 (DAU)。
* 让我们假设平均每个用户从三个不同的设备连接。
* 平均而言，如果一个用户有 200 个文件/照片，我们将拥有 1000 亿个文件。
* 假设平均文件大小为 100KB，这将为我们提供 10 PB 的总存储空间。

```
100B * 100KB => 10PB
```

* 我们还假设我们每分钟将有 100 万个活动连接。

| 总用户数    | 5亿    |
| ------- | ----- |
| 每个用户文件数 | 200   |
| 平均文件大小  | 100KB |
| 总文件数    | 1000亿 |
| 总存储量    | 10PB  |

### 5、高层次设计（High level design ，HLD）

用户将指定一个文件夹作为其设备上的工作区。放置在此文件夹中的任何文件/照片/文件夹都会上传到云端，并且无论何时修改或删除文件，都会以相同的方式反映在云存储中。用户可以在他们的所有设备上指定类似的工作空间，并且在一台设备上进行的任何修改都将传播到所有其他设备，以便在任何地方都具有相同的工作空间视图。

在高层次上，我们需要存储文件及其元数据信息，如文件名、文件大小、目录等，以及与谁共享此文件。因此，我们需要一些可以帮助客户端将文件上传/下载到云存储的服务器，以及一些可以方便更新有关文件和用户的元数据的服务器。我们还需要一些机制来在发生更新时通知所有客户端，以便他们可以同步他们的文件。

如下图所示，块服务器将与客户端一起从云存储上传/下载文件，元数据服务器将在 SQL 或 NoSQL 数据库中更新文件的元数据。同步服务器将处理通知所有客户端有关同步的不同更改的工作流。

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption><p>高层次设计（High level design ，HLD）</p></figcaption></figure>

### 6、组件设计

让我们一一浏览我们系统的主要组件：

#### a、客户端

客户端应用程序监视用户机器上的工作区文件夹，并将其中的所有文件/文件夹与远程云存储同步。客户端应用程序将与存储服务器一起上传、下载和修改实际文件到后端云存储。客户端还与远程同步服务交互以处理任何文件元数据更新，例如文件名、大小、修改日期等的更改。

以下是客户端的一些基本操作：

1. 上传和下载文件。
2. 检测工作区文件夹中的文件更改。
3. 处理由于离线或并发更新引起的冲突。

**我们如何有效地处理文件传输？**如上所述，我们可以将每个文件分成更小的块，以便我们只传输那些被修改的块而不是整个文件。假设我们将每个文件分成固定大小的 4MB 块。我们可以基于

1. 我们在云中使用的存储设备来优化空间利用率和每秒输入/输出操作 (IOPS)
2. 网络带宽
3. 存储中的平均文件大小等，静态计算什么是最佳块大小。在我们的元数据中，我们还应该记录每个文件和构成它的块。

**我们应该与客户保留一份元数据的副本吗？**保留元数据的本地副本不仅使我们能够进行离线更新，而且还节省了大量更新远程元数据的往返行程。

**客户端如何有效地监听其他客户发生的变化？** 一种解决方案可能是客户端定期检查服务器是否有任何更改。这种方法的问题在于，我们在本地反映更改时会有延迟，因为客户端会定期检查更改，而服务器会在有更改时通知。如果客户端频繁检查服务器是否有变化，不仅会浪费带宽，因为服务器大部分时间都必须返回空响应，而且还会使服务器保持忙碌状态。以这种方式提取信息是不可扩展的。

解决上述问题的方法可能是使用 HTTP 长轮询。通过长轮询，客户端从服务器请求信息，期望服务器可能不会立即响应。如果在收到轮询时服务器没有新的数据给客户端，服务器不会发送空响应，而是保持请求打开并等待响应信息可用。一旦它确实有新信息，服务器立即向客户端发送一个 HTTP/S 响应，完成打开的 HTTP/S 请求。收到服务器响应后，客户端可以立即发出另一个服务器请求以进行未来更新。

基于以上考虑，我们可以将我们的客户端分为以下四个部分：

1. **内部元数据数据库**将跟踪所有文件、块、它们的版本以及它们在文件系统中的位置。
2. **Chunker**会将文件分割成更小的块，称为块。它还将负责从其块中重建文件。我们的分块算法会检测文件中被用户修改的部分，只将这些部分传输到云存储；这将为我们节省带宽和同步时间。
3. **Watcher**将监视本地工作空间文件夹，并将用户执行的任何操作（例如，当用户创建、删除或更新文件或文件夹时）通知Indexer（如下所述）。Watcher 还监听同步服务广播的其他客户端上发生的任何更改。
4. **Indexer**将处理从 Watcher 接收到的事件，并使用有关已修改文件块的信息更新内部元数据数据库。一旦块成功提交/下载到云存储，索引器将与远程同步服务通信，将更改广播到其他客户端并更新远程元数据数据库。

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption><p>客户端设计</p></figcaption></figure>

###

**客户端应该如何处理慢速服务器？** 如果服务器忙/没有响应，客户端应该以指数方式回退。这意味着，如果服务器响应太慢，客户端应该延迟重试，并且这种延迟应该成倍增加。

**移动客户端是否应该立即同步远程更改？**与桌面或 Web 客户端不同，移动客户端通常按需同步以节省用户的带宽和空间。

#### b、元数据数据库

元数据数据库负责维护关于文件/块、用户和工作空间的版本和元数据信息。元数据数据库可以是一个关系型数据库，如MySQL，或者是一个NoSQL数据库服务，如DynamoDB。无论数据库的类型如何，同步服务应该能够使用数据库提供一致的文件视图，特别是在多个用户同时处理同一个文件的情况下。由于NoSQL数据存储不支持ACID属性，有利于可扩展性和性能，如果我们选择这种数据库，我们需要在同步服务的逻辑中以编程方式纳入对ACID属性的支持。然而，使用关系型数据库可以简化同步服务的实现，因为它们天生支持ACID属性。

元数据数据库应存储有关以下对象的信息：

1. 块
2. 文件
3. 用户
4. 设备
5. 工作区（同步文件夹）

#### c、同步服务

同步服务是处理客户端进行的文件更新并将这些更改应用到其他订阅客户端的组件。它还将客户端的本地数据库与存储在远程元数据数据库中的信息同步。同步服务是系统架构中最重要的部分，因为它在管理元数据和同步用户文件方面发挥着关键作用。桌面客户端与同步服务通信以从云存储获取更新或将文件和更新发送到云存储以及可能的其他用户。如果客户端离线一段时间，它会在新更新上线后立即轮询系统。当同步服务收到更新请求时，它会检查元数据数据库的一致性，然后继续更新。

同步服务的设计应使其在客户端和云存储之间传输更少的数据，以实现更好的响应时间。为满足此设计目标，同步服务可以使用差分算法来减少需要同步的数据量。我们可以只传输文件的两个版本之间的差异，而不是将整个文件从客户端传输到服务器，反之亦然。因此，仅传输已更改的文件部分。这也减少了最终用户的带宽消耗和云数据存储。如上所述，我们将文件分成 4MB 的块，并且只传输修改过的块。服务器和客户端可以计算一个散列（例如，SHA-256）来查看是否更新一个块的本地副本。在服务器上，如果我们已经有一个具有相似哈希的块（甚至来自另一个用户），我们不需要创建另一个副本，我们可以使用相同的块。这将在后面的重复数据删除中详细讨论。

为了能够提供高效且可扩展的同步协议，我们可以考虑在客户端和同步服务之间使用通信中间件。消息传递中间件应提供可扩展的消息队列和更改通知，以支持使用拉或推策略的大量客户端。这样，多个 Synchronization Service 实例可以接收来自全局请求[队列](https://en.wikipedia.org/wiki/Message\_queue)的请求，并且通信中间件将能够平衡其负载。

#### d、消息队列

我们架构的一个重要部分是消息传递中间件，它应该能够处理大量请求。支持客户端和同步服务之间基于异步消息的通信的可扩展消息队列服务最适合我们应用程序的要求。消息队列服务支持分布式系统组件之间的异步和松耦合的基于消息的通信。消息队列服务应该能够有效地将任意数量的消息存储在一个高度可用、可靠且可扩展的队列中。

我们架构的一个重要部分是一个消息传递中间件，它应该能够处理大量的请求。一个可扩展的消息队列服务，支持客户端和同步服务之间基于消息的异步通信，最符合我们应用的要求。消息队列服务基于消息的通信，使分布式系统的组件之间是异步和松耦合的。消息队列服务应该能够在一个高可用、可靠和可扩展的队列中有效地存储任何数量的消息。

消息队列服务将在我们的系统中实现两种类型的队列。请求队列是一个全局队列，所有客户端都将共享它。客户端更新元数据数据库的请求将首先发送到请求队列，同步服务将从那里更新元数据。对应于各个订阅客户端的响应队列负责将更新消息传递给每个客户端。由于客户端收到消息后会从队列中删除，因此我们需要为每个订阅的客户端创建单独的响应队列以共享更新消息。

<figure><img src="../.gitbook/assets/image (2) (2).png" alt=""><figcaption><p>消息队列</p></figcaption></figure>

#### e、云/快存储

[云/块存储](https://cloudacademy.com/blog/object-storage-block-storage/)存储用户上传的文件块。客户端直接与存储交互以从中发送和接收对象。元数据与存储的分离使我们能够使用云中或内部的任何存储。

<figure><img src="../.gitbook/assets/image (11) (1).png" alt=""><figcaption><p>组件设计</p></figcaption></figure>

### 7、文件处理工作流程

下面的序列显示了当客户端 A 更新与客户端 B 和 C 共享的文件时应用程序组件之间的交互，因此它们也应该收到更新。如果其他客户端在更新时不在线，消息队列服务会将更新通知保存在单独的响应队列中，直到它们稍后上线。

1. 客户端 A 将块上传到云存储。
2. 客户端 A 更新元数据并提交更改。
3. 客户 A 得到确认，并向客户 B 和 C 发送有关更改的通知。
4. 客户端 B 和 C 接收元数据更改并下载更新的块。

### 8、重复数据删除

重复数据删除是一种用于消除数据重复副本以提高存储利用率的技术。它还可以应用于网络数据传输，以减少必须发送的字节数。对于每个新传入的块，我们可以计算它的散列并将该散列与现有块的所有散列进行比较，以查看我们的存储中是否已经存在相同的块。

我们可以在我们的系统中通过两种方式实现重复数据删除：

#### a、**后处理重复数据删除**

使用后处理重复数据删除，新块首先存储在存储设备上，然后一些进程分析数据以查找重复数据。好处是客户端在存储数据之前无需等待哈希计算或查找完成，从而确保存储性能不会下降。这种方法的缺点是

1. 我们将不必要地存储重复数据，尽管在短时间内。
2. 传输重复数据会消耗带宽。

#### b、**在线重复数据删除**

可以在客户端在其设备上输入数据时实时完成重复数据删除哈希计算。如果我们的系统识别出它已经存储的块，则只会在元数据中添加对现有块的引用，而不是该块的完整副本。这种方法将为我们提供最佳的网络和存储使用。

### 9、元数据分区

为了横向扩展元数据数据库，我们需要对其进行分区，以便它可以存储有关数百万用户和数十亿文件/块的信息。我们需要提出一个分区方案，将我们的数据划分并存储在不同的数据库服务器中。

#### a、垂直分区

我们可以对数据库进行分区，以便将与某一特定功能相关的表存储在一台服务器上。例如，我们可以将所有用户相关的表存储在一个数据库中，将所有文件/块相关的表存储在另一个数据库中。尽管这种方法实现起来很简单，但它存在一些问题：

1. 我们还会有规模问题吗？如果我们有数万亿块要存储，而我们的数据库无法支持存储如此大量的记录怎么办？我们将如何进一步划分这些表？
2. 连接两个单独数据库中的两个表可能会导致性能和一致性问题。我们必须多久加入一次用户表和文件表？

#### b、基于范围的分区

如果我们根据文件路径的第一个字母将文件/块存储在单独的分区中怎么办？在这种情况下，我们将所有以字母“A”开头的文件保存在一个分区中，将那些以字母“B”开头的文件保存到另一个分区中，依此类推。这种方法称为基于范围的分区。我们甚至可以将某些不太频繁出现的字母组合到一个数据库分区中。我们应该静态地提出这种分区方案，以便我们始终可以以可预测的方式存储/查找文件。

这种方法的主要问题是它可能导致服务器不平衡。例如，如果我们决定将所有以字母“E”开头的文件放入数据库分区，然后我们意识到我们有太多以字母“E”开头的文件，以至于我们无法容纳它们到一个数据库分区。

#### c、基于散列的分区

在这个方案中，我们获取我们正在存储的对象的散列，并根据这个散列计算出这个对象应该去的 DB 分区。在我们的例子中，我们可以使用我们正在存储的文件对象的“FileID”的哈希值来确定文件将被存储的分区。我们的散列函数会将对象随机分布到不同的分区中，例如，我们的散列函数总是可以将任何 ID 映射到 \[1…256] 之间的数字，而这个数字将是我们将存储对象的分区。

这种方法仍然会导致分区过载，这可以通过使用[Consistent Hashing](https://www.educative.io/collection/page/5668639101419520/5649050225344512/5709068098338816)来解决。

### 10、缓存

我们的系统中可以有两种缓存。为了处理热文件/块，我们可以为块存储引入缓存。我们可以使用像[Memcached](https://en.wikipedia.org/wiki/Memcached)这样的现成解决方案，它可以使用各自的 ID/Hash 存储整个块，并且块服务器在访问块存储之前可以快速检查缓存是否具有所需的块。根据客户的使用模式，我们可以确定我们需要多少缓存服务器。高端商用服务器可以有144GB内存；一台这样的服务器可以缓存 36K 块。

**哪种缓存替换策略最适合我们的需求？**当缓存已满时，我们想用更新/更热的块替换一个块，我们将如何选择？最近最少使用 (LRU) 可能是我们系统的合理策略。在此策略下，我们首先丢弃最近最少使用的块。加载类似地，我们可以为元数据数据库缓存。

### 11、负载均衡器（LB）

我们可以在系统的两个地方添加负载均衡层：

1. 客户端和块服务器之间
2. 客户端和元数据服务器之间。

最初，可以采用一种简单的循环方法，在后端服务器之间平均分配传入的请求。这个 LB 实现简单，不会引入任何开销。这种方法的另一个好处是，如果服务器挂了，LB 将把它从轮换中取出，并停止向它发送任何流量。Round Robin LB 的一个问题是，它不会考虑服务器负载。如果服务器过载或速度慢，LB 将不会停止向该服务器发送新请求。为了解决这个问题，可以放置一个更智能的 LB 解决方案，它会定期向后端服务器查询其负载并据此调整流量。

### 12、安全、权限和文件共享

用户在将文件存储在云中时主要关心的问题之一是数据的隐私和安全性，尤其是因为在我们的系统中，用户可以与其他用户共享他们的文件，甚至可以将其公开以与所有人共享。为了解决这个问题，我们将在元数据数据库中存储每个文件的权限，以反映任何用户可见或可修改的文件。
