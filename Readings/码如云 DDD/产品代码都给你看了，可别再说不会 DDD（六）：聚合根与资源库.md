---
title: 产品代码都给你看了，可别再说不会 DDD（六）：聚合根与资源库
author:
  - "[滕云](https://www.cnblogs.com/davenkin/)"
date created: 2025-04-16 14:50:37
category: 计算机科学
tags:
  - "#领域驱动设计"
  - "#软件架构"
  - "#软件开发"
  - "#编程实践"
  - "#代码优化"
url:
  - https://docs.mryqr.com/ddd-aggregate-root-and-repository/
description: 本文详细介绍了领域驱动设计（DDD）中的核心概念——聚合根，以及与之密切相关的资源库（Repository）。文章深入探讨了聚合根的定义、设计原则（如内聚性、对外黑盒、不变条件），并通过实际代码示例展示了如何在Java中实现聚合根。此外，还讨论了跨聚合根用例的处理方式，以及资源库在DDD中的作用和实现方法，旨在帮助读者更好地理解和应用DDD进行软件开发。
status: Finished
---

# 聚合根与资源库

在上一篇 [请求处理流程](https://docs.mryqr.com/ddd-request-process-flow) 中我们讲到，领域模型是 DDD 的核心，而聚合根又是领域模型的核心。从某种意义上讲，DDD 中其它组件均可看作是对聚合根的支撑或辅助。在本文中，我们将对聚合根以及与之密切相关的资源库 (Repository) 做详细的讲解。

## 聚合根是什么

在 [DDD 概念大白话](https://docs.mryqr.com/ddd-in-plain-words) 一文中，我们讲到了 “什么是聚合根”，这里再重复一下。聚合根中的 “聚合” 即 “高内聚，低耦合” 中的 “内聚” 之意；而 “根” 则是 “根部” 的意思，也即聚合根是一种统领式的存在。事实上，并不存在一个教科书式的对聚合根的理论定义，你可以将聚合根理解为一个系统中最重要最显著的那些名词，这些名词是其所在的软件系统之所以存在的原因。为了给你一个直观的理解，以下是几个聚合根的例子：

- 在一个电商系统中，一个订单（Order）对象表示一个聚合根
- 在一个 CRM 系统中，一个客户（Customer）对象表示一个聚合根
- 在一个银行系统中，一次交易（Transaction）对象表示一个聚合根

你可能会问，软件中的概念已经很多了，为什么还要搞出个聚合根的概念？我们认为这里至少有 2 点原因：

1. 聚合根遵循了软件中 “高内聚，低耦合” 的基本原则
2. 聚合根体现了一种模块化的原则，模块化思想是被各个行业所证明的可以降低系统复杂度的一种思想。所谓的 DDD 是 “软件核心复杂性应对之道”，也即这个意思，它将软件系统在人脑中所呈现地更加有序和简单，让人可以更好地理解和管控软件系统。

在实际项目中识别聚合根时，我们需要对业务有深入的了解，因为只有这样你才知道到底哪些业务逻辑是内聚在一起的。这也是我们一直建议程序员和架构师们不要一味地埋头于技术而要多关注业务的原因。

事实上，如果让一个从来没有接触过 DDD 的人来建模，十有八九也能设计出上面的订单、客户和交易对象出来。没错，DDD 绝非什么颠覆式的发明，依然只是在前人基础上的一种进步而已，这种进步更多的体现在一些设计原则上，对此我们将在下文进行详细阐述。

## 聚合根基类

在代码实现层面，一般的实践是将所有的聚合根都继承自一个公共基类 `AggregateRoot`：

```java
//AggregateRoot

@Getter
public abstract class AggregateRoot implements Identified {
    private String id;//聚合根ID
    private String tenantId;//租户ID
    private Instant createdAt;//创建时间
    private String createdBy;//创建人的MemberId
    private String creator;//创建人姓名
    private Instant updatedAt;//更新时间
    private String updatedBy;//更新人MemberId
    private String updater;//更新人姓名
    private List<DomainEvent> events;//临时存放领域事件
    private LinkedList<OpsLog> opsLogs;//操作日志

    @Version
    @Getter(PRIVATE)
    private Long _version;//版本号，实现乐观锁

    //...此处省略了AggregateRoot中行为方法

    @Override
    public String getIdentifier() {
        return id;
    }
}
```

> 源码出处：[com/mryqr/core/common/domain/AggregateRoot.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/common/domain/AggregateRoot.java)

在 `AggregateRoot` 中，包含聚合根 ID（`id`）、创建信息 (`createdAt` 和 `createdBy`) 和更新信息（`updatedAt` 和 `updatedBy`）等数据。租户 ID（`tenantId`）用于标定聚合根所在的租户（码如云是一个多租户系统）。另外，`events` 用于临时性存放聚合根中所产生的领域事件，我们将在 [领域事件](https://docs.mryqr.com/ddd-domain-events) 一文中对此所详细解释。

实际的聚合根继承自 `AggregateRoot`，例如，在 [码如云](https://www.mryqr.com/) 中，**分组**（Group）聚合根的实现如下：

```java
@Getter
@Document(GROUP_COLLECTION)
@TypeAlias(GROUP_COLLECTION)
@NoArgsConstructor(access = PRIVATE)
public class Group extends AggregateRoot {
    private String name;//名称
    private String appId;//所在的app
    private List<String> managers;//管理员
    private List<String> members;//普通成员
    private boolean archived;//是否归档
    private String customId;//自定义编号
    private boolean active;//是否启用
    private String departmentId;//由哪个部门同步而来
    
    //...此处省略了Group的行为方法
}
```

> 源码出处：[com/mryqr/core/group/domain/Group.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/domain/Group.java)

## 聚合根基本原则

从上面的代码例子可以看出，聚合根只是普通的 Java 对象而已，真正使之成为聚合根的是一些特定的设计原则。

### 内聚性原则

这个原则不用我们再细讲了吧，估计你在大学里就学过，只举个例子，对于上面的分组 `Group` 对象来说，管理员 `managers`、普通成员 `members` 以及启用标志 `active` 均是 `Group` 不可分割的属性，这些属性独立于 `Group` 是无法存在的。

### 对外黑盒原则

对外黑盒原则讲的是，聚合根的外部（也即聚合根的调用方或客户方）不需要关心聚合根内部的实现细节，而只需要通过调用聚合根向外界暴露的共有业务方法即可。具体表现为，外部对聚合根的调用只能通过根对象完成，而不能调用聚合根内部对象上的方法。举个例子，在码如云中，管理员可以向**分组** (Group) 中添加成员，具体的实现代码如下：

```java
//Group

public void addMembers(List<String> memberIds, User user) {
    if (isSynced()) {
        throw new MryException(GROUP_SYNCED,
                "无法添加成员，已设置从部门同步。",
                "groupId", this.getId());
    }

    this.members = concat(members.stream(), memberIds.stream())
            .distinct()
            .collect(toImmutableList());
    
    addOpsLog("设置成员", user);
}
```

> 源码出处：[com/mryqr/core/group/domain/Group.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/domain/Group.java)

这里，外部在向分组中添加成员时，需要调用 `Group` 上的 `addMembers()` 方法，该方法知道将 `memberIds` 添加到自身的 `members` 字段中，这个过程对外部是不可见的。与之相对的另一种方式是，外部调用法先拿到 `Member` 的 `members` 引用，然后由外部自行向 `members` 中添加 `memberIds`：

```java
//外部调用方

@Transactional
public void addGroupMembers(String groupId, List<String> memberIds, User user) {
    Group group = groupRepository.byIdAndCheckTenantShip(groupId, user);

    if (group.isSynced()) {
        throw new MryException(GROUP_SYNCED,
                "无法添加成员，已设置从部门同步。",
                "groupId", group.getId());
    }

    List<String> members = group.getMembers();
    members.addAll(memberIds);
    groupRepository.save(group);

    log.info("Added members{} to group[{}].", memberIds, groupId);
}
```

> 源码出处：[com/mryqr/core/group/command/GroupCommandService.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/command/GroupCommandService.java)

这种方式是一种反模式，存在以下缺点：

- 外部需要了解 `Group` 的内部结构，背离了对外黑盒原则，本例中，外部通过 `group.getMembers()` 获取到了 `Group` 内部的 `members` 属性
- 聚合根内部的业务逻辑泄漏到了外部，背离了内聚性原则，本例中，对 `group.isSynced()` 的调用原本应该放在 `Group` 中的，结果却由外部承担了该职责

在对外黑盒原则的指导下，聚合根自然形成了一个边界，它站在这个边界上向外声明：“我所包围着的内部的所有均由我负责，如果谁想访问我的内部，直接访问是被禁止的，只能通过我这个 “根” 来访问。”

### 不变条件原则

不变条件 (Invariants) 表示聚合根需要保证其内部在任何时候均处于一种合法的状态（也即数据一致性需要得到保证），一个常见的例子是订单 (Order) 中有订单项 (OrderItem) 和订单价格 (Price)，当订单项发生变化时，其价格应该随之发生变化，并且这两种变化应该在订单的同一个业务方法中完成。这一点是好理解的，既然聚合根对外是一个黑盒，那么外界便不会负责给你聚合根擦屁股，你聚合根自己需要保证自身的正确性。

在码如云中，应用管理员可以向**分组** (Group) 中添加分组管理员。这其中有层隐含意思是，既然分组管理员也是分组成员，那么在添加分组管理员的同时需要一并将其添加到分组成员中，具体实现代码如下：

```java
//Group

public void addManager(String memberId, User user) {
    if (!this.members.contains(memberId)) {
        this.members = concat(members.stream(), Stream.of(memberId))
                .distinct()
                .collect(toImmutableList());
    }

    this.managers = concat(this.managers.stream(), Stream.of(memberId))
            .distinct()
            .collect(toImmutableList());
    
    raiseEvent(new GroupManagersChangedEvent(this.getId(), this.getAppId(), user));
    
    addOpsLog("添加管理员", user);
}
```

> 源码出处：[com/mryqr/core/group/domain/Group.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/domain/Group.java)

在本例的添加分组管理员 `addManager()` 方法中，我们除了向 `managers` 中添加成员外，还保证了该成员也出现在 `members` 中。这里的 “分组管理员也是分组成员” 即是一种不变条件，我们需要在聚合根内部保证不变条件不被破坏，因为不变条件往往意味着核心的业务逻辑。

### 通过 ID 引用其他聚合根原则

当一个聚合根需要引用另一个聚合根时，并不需要维持对另一聚合根的整体引用，而是只需通过 ID 进行引用即可。这个原则的出发点是：聚合根和聚合根之间是一种平级关系，并不是隶属关系，每个聚合根本身是一个相对独立的模块，其与其他聚合根的关系应该通过 ID 这种松耦合的方式进行引用，如果整体引用则更像是一种包含关系。

在码如云中，**分组** (Group) 通过 `appId` 引用其所属的**应用** (App)，通过 `departmentId` 引用所同步的**部门** (Department)，而在 `managers` 和 `members` 字段中，则是以 `memberId` 引用相应**成员** (Member)：

```java
@Getter
@Document(GROUP_COLLECTION)
@TypeAlias(GROUP_COLLECTION)
@NoArgsConstructor(access = PRIVATE)
public class Group extends AggregateRoot {
    private String name;//名称
    private String appId;//所在的app
    private List<String> managers;//管理员
    private List<String> members;//普通成员
    private boolean archived;//是否归档
    private String customId;//自定义编号
    private boolean active;//是否启用
    private String departmentId;//由哪个部门同步而来
   
    //...省略其他代码
}
```

> 源码出处：[com/mryqr/core/group/domain/Group.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/domain/Group.java)

### 与基础设施无关原则

既然整个领域模型与基础设施无关，那么位于领域模型之内的聚合根自然也不能与基础设施相关，这样好处是将业务复杂度与技术复杂度解耦开来，让业务模型可以独立于技术设施而完成自身的演变。比如，假设一个项目需要从 Spring 框架迁移到 Guice 框架，此时如果能够保证领域模型与基础设施的无关性，那么对领域模型的迁移过程讲变得非常简单，基本上无需修改任何代码直接拷贝到新的项目中即可。

事实上，码如云尚未完全做到这一点，从上面的例子中可以看到，`AggregateRoot` 和 `Group` 对 Spring 框架中的 `@Version`、`@Document` 和 `@TypeAlias`3 个与持久化相关的注解存在引用。如需解决这个问题，可以考虑在领域模型之外另建专门用于数据库访问的 [持久化对象 (Persistence Model)](https://khorikov.org/posts/2020-04-20-when-do-you-need-persistence-model/)。但是，引入持久化对象是有成本的，比如需要维护领域对象与持久化对象之间的相互转化等。在码如云，我们选择了妥协，一方面考虑到持久化对象的成本，另一方面我们也预见在将来要迁移出 Spring 框架的几率是非常小的。不过，除了前面提到的 3 个注解之外，码如云中的聚合根可以做到对基础设施没有任何其他引用。关于持久化对象，在 [Stackoverflow](https://stackoverflow.com/questions/14024912/ddd-persistence-model-and-domain-model) 上有过非常有意义的讨论，读者可自行阅览。

## 跨聚合根用例

通常来讲，一个业务用例只会操作一个（或一种）聚合根。但有时，一个业务用例可能会导致多个（或多种）聚合根对象的更新，此时可分两种情况：

1. 如果聚合根位于不同的进程空间（比如不同的微服务）中，那么解决方式一是可以使用 [事件驱动架构 (EDA)](https://aws.amazon.com/event-driven-architecture/)，二是通过全局事务（比如 JTA）完成。基于全局事务的性能和效率低下等问题，DDD 社区一般建议采用事件驱动架构，即在一个进程空间中只对其包含的聚合根进行操作，然后通过向其他进程空间发送事件通知的方式，使得其他进程空间做相应的聚合根更新。
2. 如果聚合根位于同一个进程空间，此时依然可以选择事件驱动架构，但是另一种更简单实用的方式是直接同时更新多个聚合根，毕竟此时对所有聚合根的更新均处于同一个本地事务中。

> 所谓全局事务，是和本地事务相对的一个概念。本地事务保证一个数据库内的 ACID 特性，而全局事务保证跨数据库的 ACID。而 JTA（Java Transaction API），通常我们说「声明式事务」，它允许 Java 应用程序以平台无关的方式访问事务管理器，并参与到全局事务中。之所以说 JTA 是全局事务，是因为 JTA 的设计目标就是为了解决跨多个资源（例如多个数据库）的事务一致性问题，也就是我们常说的分布式事务或全局事务。
>
> JTA 允许一个事务跨越多个数据库或其他支持 XA 协议的资源管理器。这意味着一个事务可以同时更新多个数据库，并且保证这些更新要么全部成功，要么全部失败。这与本地事务只能在一个数据库内保证 ACID 特性不同。
>
> 简而言之按分布式事务看就行了。

[码如云](https://www.mryqr.com/) 是一个单体系统，因此属于以上的第 2 种情况，我们根据聚合根之间的业务紧密程度的不同，在有些场景下选择了同时更新多个聚合根，在另一些场景下则选择通过事件驱动机制解决。比如，在 “创建实例” 的用例中，除了创建**实例** (QR) 之外，还需要创建该实例对应的**码牌** (Plate)，由于 “有实例就必有码牌”，因此它们之间是紧密联系的，故在码如云中我们选择了在同一个本地事务中同时更新实例和码牌：

```java
//QrCommandService
    
@Transactional
public CreateQrResponse createQr(CreateQrCommand command, User user) {
    String name = command.getName();
    String groupId = command.getGroupId();

    Group group = groupRepository.cachedByIdAndCheckTenantShip(groupId, user);
    String appId = group.getAppId();
    App app = appRepository.cachedById(appId);

    PlatedQr platedQr = qrFactory.createPlatedQr(name, group, app, user);
    QR qr = platedQr.getQr();
    Plate plate = platedQr.getPlate();

    //同时保存QR和Plate
    qrRepository.save(qr);
    plateRepository.save(plate);

    log.info("Created qr[{}] of group[{}] of app[{}].",
            qr.getId(), groupId, appId);

    return CreateQrResponse.builder()
            .qrId(qr.getId())
            .plateId(plate.getId())
            .groupId(groupId)
            .appId(appId)
            .build();
}
```

> 源码出处：[com/mryqr/core/group/command/GroupCommandService.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/command/GroupCommandService.java)

可以看到，在用例方法 `createQr()` 中，我们先后调用 `qrRepository.save(qr)` 和 `plateRepository.save(plate)` 分别完成了对 `QR` 和 `Plate` 的持久化。

如果你希望了解事件驱动架构相关的知识，请参考本系列的 [领域事件](https://docs.mryqr.com/ddd-domain-events) 一文。

## 资源库

在 DDD 中，**资源库** (Repository) 以聚合根为单位完成对数据库的访问。这里的重点是 “以聚合根为单位”，也即只有聚合根才配得上拥有资源库（毕竟在 DDD 中大家都是围绕着聚合根转的嘛），其他对象（比如非聚合根实体）是没有对应资源库的，这也是资源库和 [DAO](https://www.baeldung.com/java-dao-pattern) 最大的区别。在编码实现时，资源库方法所接受的参数和返回的数据都应该是聚合根对象，例如，在码如云中，**成员** (Member) 聚合根对应的资源库定义如下：

```java
public interface MemberRepository {
    Member byId(String id); //返回聚合根

    Optional<Member> byIdOptional(String id); //返回聚合根

    Member byIdAndCheckTenantShip(String id, User user); //返回聚合根

    void save(Member member); //聚合根作为参数

    void delete(Member member); //聚合根作为参数
}
```

> 源码出处：[com/mryqr/core/member/domain/MemberRepository.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/member/domain/MemberRepository.java)

行业中这么一个现象，很多程序员在面对一个新的业务需求时，首先想到的是如何设计数据库的表结构，然后再编写业务代码。在 DDD 中，这是一种反模式，既然是 “领域驱动”，那么我们首先应该关心的是如何业务建模，而不是数据库建模。事实上，正如 Robert C. Martin 在《整洁架构》一书中所说，数据库只是一个实现细节而已，不应该成为软件建模的主体。

资源库的作用，在于它在业务复杂度和技术复杂度之间做了一层很好的隔离，让我们可以独立地看待软件的业务模型而不受技术设施的影响。从本质上讲，资源库做的事情只是实现数据在内存和磁盘之间相互传输而已。在编程实现业务逻辑的时候，我们只需关心内存中的那个聚合根对象即可，当聚合根对象的状态由于业务操作发生了改变之后，再调用资源库将新的聚合根状态同步到磁盘中完成持久化，在调用时我们假设并相信资源库一定可以完成其自身的使命。

```java
@Transactional
public void addGroupManager(String groupId, String memberId, User user) {
    Group group = groupRepository.byIdAndCheckTenantShip(groupId, user);

    group.addManager(memberId, user);
    
    groupRepository.save(group);
    
    log.info("Added manager[{}] to group[{}].", memberId, groupId);
}
```

> 源码出处：[com/mryqr/core/group/command/GroupCommandService.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/group/command/GroupCommandService.java)

在上例的 “向分组中添加管理员” 用例中，首先通过资源库 `GroupRepository` 的 `byIdAndCheckTenantShip()` 方法得到聚合根 `Group` 对象，然后再完成后续操作。这里的 `addGroupManager()` 无需知道 `Group` 是如何加载的，甚至不用知道后台使用的是 MySQL 还是 MongoDB 或是其他，反正通过调用 `GroupRepository.byIdAndCheckTenantShip()` 可以得到一个完整合法的 `Group` 对象即可。

在资源库中，最重要的方法有以下 3 个：

```java
public interface GroupRepository {
    
    Group byId(String id);
    
    void save(Group group);
    
    void delete(Group group);
}
```

> 源码出处：[com/mryqr/core/member/domain/MemberRepository.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/member/domain/MemberRepository.java)

其中，`byId()` 用于根据 ID 获取指定聚合根，`save()` 用于保存聚合根，`delete()` 则用于删除聚合根。除此之外，资源库中还可以包含更多的查询方法，比如在 `GroupRepository` 中还包含以下方法：

```java
//根据部门ID查找分组
List<Group> byDepartmentId(String departmentId);

//根据ID查找分组，返回Optional
Optional<Group> byIdOptional(String id);

//根据ID查找分组，同时检查租户
Group byIdAndCheckTenantShip(String id, User user);
```

> 源码出处：[com/mryqr/core/member/domain/MemberRepository.java](https://github.com/mryqr-com/mry-backend/blob/main/src/main/java/com/mryqr/core/member/domain/MemberRepository.java)

需要注意的是，这里的查询方法指的是在实现业务逻辑的过程中需要做的查询操作，并不是为了前端显示那种纯粹的查询，因为纯粹的查询操作不见得一定要放到资源库中，而是可以作为一个单独的关注点通过 [CQRS](https://docs.mryqr.com/ddd-cqrs) 解决。

在 DDD 项目中，通常将资源库分为接口类和实现类，将接口类放置在领域模型 `domain` 包中，而将实现类放置在基础设施 `infrastructure` 包中，这种做法有 2 点好处：

1. 通过依赖反转，使得领域模型不依赖于基础设施
2. 实现资源库的可插拔性，比如未来需要从 MongoDB 迁移到 MySQL，那么只需创建新的实现类即可

# 总结

在本文中，我们讲到了作为 DDD 核心的聚合根的设计原则及实现，其中包含内聚原则、对外黑盒原则和不变条件原则等。此外，我们也对与聚合根密切相关的资源库做了讲解。在下一篇 [实体与值对象](https://docs.mryqr.com/ddd-entity-and-value-object) 中，我们将讲到实体和值对象之间的区别，以及各自的典型编码实践。
