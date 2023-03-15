# 【NO.365】设计模式二三事

## 1.引言

话说这是在程序员世界里一对师徒的对话：

“老师，我最近在写代码时总感觉自己的代码很不优雅，有什么办法能优化吗？”

“嗯，可以考虑通过教材系统学习，从注释、命名、方法和异常等多方面实现整洁代码。”

“然而，我想说的是，我的代码是符合各种编码规范的，但是从实现上却总是感觉不够简洁，而且总是需要反复修改！”学生小明叹气道。

老师看了看小明的代码说：“我明白了，这是系统设计上的缺陷。总结就是抽象不够、可读性低、不够健壮。”

“对对对，那怎么能迅速提高代码的可读性、健壮性、扩展性呢？”小明急不可耐地问道。

老师敲了敲小明的头：“不要太浮躁，没有什么方法能让你立刻成为系统设计专家。但是对于你的问题，我想**设计模式**可以帮到你。”

“设计模式？”小明不解。

“是的。”老师点了点头，“世上本没有路，走的人多了，便变成了路。在程序员的世界中，本没有设计模式，写代码是人多了，他们便总结出了一套能提高开发和维护效率的套路，这就是设计模式。设计模式不是什么教条或者范式，它可以说是一种在特定场景下普适且可复用的解决方案，是一种可以用于提高代码可读性、可扩展性、可维护性和可测性的最佳实践。”

“哦哦，我懂了，那我应该如何去学习呢？”

“不急，接下来我来带你慢慢了解设计模式。”

## 2.奖励的发放策略

第一天，老师问小明：“你知道活动营销吗？”

“这我知道，活动营销是指企业通过参与社会关注度高的已有活动，或整合有效的资源自主策划大型活动，从而迅速提高企业及其品牌的知名度、美誉度和影响力，常见的比如有抽奖、红包等。”

老师点点头：“是的。我们假设现在就要做一个营销，需要用户参与一个活动，然后完成一系列的任务，最后可以得到一些奖励作为回报。活动的奖励包含美团外卖、酒旅和美食等多种品类券，现在需要你帮忙设计一套奖励发放方案。”

因为之前有过类似的开发经验，拿到需求的小明二话不说开始了编写起了代码：

```
// 奖励服务
class RewardService {
    // 外部服务
    private WaimaiService waimaiService;
    private HotelService hotelService;
    private FoodService foodService;
    // 使用对入参的条件判断进行发奖
    public void issueReward(String rewardType, Object ... params) {
        if ("Waimai".equals(rewardType)) {
            WaimaiRequest request = new WaimaiRequest();
            // 构建入参
            request.setWaimaiReq(params);
            waimaiService.issueWaimai(request);
        } else if ("Hotel".equals(rewardType)) {
            HotelRequest request = new HotelRequest();
            request.addHotelReq(params);
            hotelService.sendPrize(request);
        } else if ("Food".equals(rewardType)) {
            FoodRequest request = new FoodRequest(params);
            foodService.getCoupon(request);
        } else {
           throw new IllegalArgumentException("rewardType error!");
        }
    }
}
```

小明很快写好了Demo，然后发给老师看。

“假如我们即将接入新的打车券，这是否意味着你必须要修改这部分代码？”老师问道。

小明愣了一愣，没等反应过来老师又问：”假如后面美团外卖的发券接口发生了改变或者替换，这段逻辑是否必须要同步进行修改？”

小明陷入了思考之中，一时间没法回答。

经验丰富的老师一针见血地指出了这段设计的问题：“你这段代码有两个主要问题，一是不符合**开闭原则**，可以预见，如果后续新增品类券的话，需要直接修改主干代码，而我们提倡代码应该是对修改封闭的；二是不符合**迪米特法则**，发奖逻辑和各个下游接口高度耦合，这导致接口的改变将直接影响到代码的组织，使得代码的可维护性降低。”

小明恍然大悟：“那我将各个同下游接口交互的功能抽象成单独的服务，封装其参数组装及异常处理，使得发奖主逻辑与其解耦，是否就能更具备扩展性和可维护性？”

“这是个不错的思路。之前跟你介绍过设计模式，这个案例就可以使用**策略模式**和**适配器模式**来优化。”

小明借此机会学习了这两个设计模式。首先是策略模式：

> 策略模式定义了一系列的算法，并将每一个算法封装起来，使它们可以相互替换。策略模式通常包含以下角色：
>
> - 抽象策略（Strategy）类：定义了一个公共接口，各种不同的算法以不同的方式实现这个接口，环境角色使用这个接口调用不同的算法，一般使用接口或抽象类实现。
> - 具体策略（Concrete Strategy）类：实现了抽象策略定义的接口，提供具体的算法实现。
> - 环境（Context）类：持有一个策略类的引用，最终给客户端调用。

然后是适配器模式：

> 适配器模式：将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类能一起工作。适配器模式包含以下主要角色：
>
> - 目标（Target）接口：当前系统业务所期待的接口，它可以是抽象类或接口。
> - 适配者（Adaptee）类：它是被访问和适配的现存组件库中的组件接口。
> - 适配器（Adapter）类：它是一个转换器，通过继承或引用适配者的对象，把适配者接口转换成目标接口，让客户按目标接口的格式访问适配者。

结合优化思路，小明首先设计出了策略接口，并通过适配器的思想将各个下游接口类适配成策略类：

```
// 策略接口
interface Strategy {
    void issue(Object ... params);
}
// 外卖策略
class Waimai implements Strategy {
   private WaimaiService waimaiService;
    @Override
    public void issue(Object... params) {
        WaimaiRequest request = new WaimaiRequest();
        // 构建入参
        request.setWaimaiReq(params);
        waimaiService.issueWaimai(request);
    }
}
// 酒旅策略
class Hotel implements Strategy {
   private HotelService hotelService;
    @Override
    public void issue(Object... params) {
        HotelRequest request = new HotelRequest();
        request.addHotelReq(params);
        hotelService.sendPrize(request);
    }
}
// 美食策略
class Food implements Strategy {
   private FoodService foodService;
    @Override
    public void issue(Object... params) {
        FoodRequest request = new FoodRequest(params);
        foodService.payCoupon(request);
    }
}
```

然后，小明创建策略模式的环境类，并供奖励服务调用：

```
// 使用分支判断获取的策略上下文
class StrategyContext {
    public static Strategy getStrategy(String rewardType) {
        switch (rewardType) {
            case "Waimai":
                return new Waimai();
            case "Hotel":
                return new Hotel();
            case "Food":
                return new Food();
            default:
                throw new IllegalArgumentException("rewardType error!");
        }
    }
}
// 优化后的策略服务
class RewardService {
    public void issueReward(String rewardType, Object ... params) {
        Strategy strategy = StrategyContext.getStrategy(rewardType);
        strategy.issue(params);
    }
}
```

小明的代码经过优化后，虽然结构和设计上比之前要复杂不少，但考虑到健壮性和拓展性，还是非常值得的。

“看，我这次优化后的版本是不是很完美？”小明洋洋得意地说。

“耦合度确实降低了，但还能做的更好。”

“怎么做？”小明有点疑惑。

“我问你，策略类是有状态的模型吗？如果不是是否可以考虑做成单例的？”

“的确如此。”小明似乎明白了。

“还有一点，环境类的获取策略方法职责很明确，但是你依然没有做到完全对修改封闭。”

经过老师的点拨，小明很快也领悟到了要点：“那我可以将策略类单例化以减少开销，并实现自注册的功能彻底解决分支判断。”

小明列出单例模式的要点：

> 单例模式设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
>
> 这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

最终，小明在策略环境类中使用一个注册表来记录各个策略类的注册信息，并提供接口供策略类调用进行注册。同时使用**饿汉式单例模式**去优化策略类的设计：

```
// 策略上下文，用于管理策略的注册和获取
class StrategyContext {
    private static final Map<String, Strategy> registerMap = new HashMap<>();
    // 注册策略
    public static void registerStrategy(String rewardType, Strategy strategy) {
        registerMap.putIfAbsent(rewardType, strategy);
    }
    // 获取策略
    public static Strategy getStrategy(String rewardType) {
        return registerMap.get(rewardType);
    }
}
// 抽象策略类
abstract class AbstractStrategy implements Strategy {
    // 类注册方法
    public void register() {
        StrategyContext.registerStrategy(getClass().getSimpleName(), this);
    }
}
// 单例外卖策略
class Waimai extends AbstractStrategy implements Strategy {
    private static final Waimai instance = new Waimai();
   private WaimaiService waimaiService;
    private Waimai() {
        register();
    }
    public static Waimai getInstance() {
        return instance;
    }
    @Override
    public void issue(Object... params) {
        WaimaiRequest request = new WaimaiRequest();
        // 构建入参
        request.setWaimaiReq(params);
        waimaiService.issueWaimai(request);
    }
}
// 单例酒旅策略
class Hotel extends AbstractStrategy implements Strategy {
   private static final Hotel instance = new Hotel();
   private HotelService hotelService;
    private Hotel() {
        register();
    }
    public static Hotel getInstance() {
        return instance;
    }
    @Override
    public void issue(Object... params) {
        HotelRequest request = new HotelRequest();
        request.addHotelReq(params);
        hotelService.sendPrize(request);
    }
}
// 单例美食策略
class Food extends AbstractStrategy implements Strategy {
   private static final Food instance = new Food();
   private FoodService foodService;
    private Food() {
        register();
    }
    public static Food getInstance() {
        return instance;
    }
    @Override
    public void issue(Object... params) {
        FoodRequest request = new FoodRequest(params);
        foodService.payCoupon(request);
    }
}
```

最终，小明设计完成的结构类图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVm2AEU6AQbUAyCqHgKROft5pgQOhtR90ia1fng0MAd8CLhTOWZA1J7SvIAZPTp5azBtREeQJwwrfQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)奖励发放策略_类图

如果使用了Spring框架，还可以利用Spring的Bean机制来代替上述的部分设计，直接使用`@Component`和`@PostConstruct`注解即可完成单例的创建和注册，代码会更加简洁。

至此，经过了多次讨论、反思和优化，小明终于得到了一套低耦合高内聚，同时符合开闭原则的设计。

“老师，我开始学会利用设计模式去解决已发现的问题。这次我做得怎么样？”

“合格。但是，依然要戒骄戒躁。”

## 3.任务模型的设计

“之前让你设计奖励发放策略你还记得吗？”老师忽然问道。

“当然记得。一个好的设计模式，能让工作事半功倍。”小明答道。

“嗯，那会提到了活动营销的组成部分，除了奖励之外，貌似还有任务吧。”

小明点了点头，老师接着说：“现在，我想让你去完成任务模型的设计。你需要重点关注状态的流转变更，以及状态变更后的消息通知。”

小明欣然接下了老师给的难题。他首先定义了一套任务状态的枚举和行为的枚举：

```
// 任务状态枚举
@AllArgsConstructor
@Getter
enum TaskState {
    INIT("初始化"),
    ONGOING( "进行中"),
    PAUSED("暂停中"),
    FINISHED("已完成"),
    EXPIRED("已过期")
    ;
    private final String message;
}
// 行为枚举
@AllArgsConstructor
@Getter
enum ActionType {
    START(1, "开始"),
    STOP(2, "暂停"),
    ACHIEVE(3, "完成"),
    EXPIRE(4, "过期")
    ;
    private final int code;
    private final String message;
}
```

然后，小明对开始编写状态变更功能：

```
class Task {
    private Long taskId;
    // 任务的默认状态为初始化
    private TaskState state = TaskState.INIT;
    // 活动服务
    private ActivityService activityService;
    // 任务管理器
    private TaskManager taskManager;
    // 使用条件分支进行任务更新
    public void updateState(ActionType actionType) {
        if (state == TaskState.INIT) {
            if (actionType == ActionType.START) {
                state = TaskState.ONGOING;
            }
        } else if (state == TaskState.ONGOING) {
            if (actionType == ActionType.ACHIEVE) {
                state = TaskState.FINISHED;
                // 任务完成后进对外部服务进行通知
                activityService.notifyFinished(taskId);
                taskManager.release(taskId);
            } else if (actionType == ActionType.STOP) {
                state = TaskState.PAUSED;
            } else if (actionType == ActionType.EXPIRE) {
                state = TaskState.EXPIRED;
            }
        } else if (state == TaskState.PAUSED) {
            if (actionType == ActionType.START) {
                state = TaskState.ONGOING;
            } else if (actionType == ActionType.EXPIRE) {
                state = TaskState.EXPIRED;
            }
        }
    }
}
```

在上述的实现中，小明在`updateState`方法中完成了2个重要的功能：

1. 接收不同的行为，然后更新当前任务的状态；
2. 当任务过期时，通知任务所属的活动和任务管理器。

诚然，随着小明的系统开发能力和代码质量意识的提升，他能够认识到这种功能设计存在缺陷。

“老师，我的代码还是和之前说的那样，不够优雅。”

“哦，你自己说说看有什么问题？”

“第一，方法中使用条件判断来控制语句，但是当条件复杂或者状态太多时，条件判断语句会过于臃肿，可读性差，且不具备扩展性，维护难度也大。且增加新的状态时要添加新的if-else语句，这违背了开闭原则，不利于程序的扩展。”

老师表示同意，小明接着说：“第二，任务类不够高内聚，它在通知实现中感知了其他领域或模块的模型，如活动和任务管理器，这样代码的耦合度太高，不利于扩展。”

老师赞赏地说道：“很好，你有意识能够自主发现代码问题所在，已经是很大的进步了。”

“那这个问题应该怎么去解决呢？”小明继续发问。

“这个同样可以通过设计模式去优化。首先是状态流转的控制可以使用**状态模式**，其次，任务完成时的通知可以用到**观察者模式**。”

收到指示后，小明马上去学习了状态模式的结构：

> 状态模式：对有状态的对象，把复杂的“判断逻辑”提取到不同的状态对象中，允许状态对象在其内部状态发生改变时改变其行为。状态模式包含以下主要角色：
>
> - 环境类（Context）角色：也称为上下文，它定义了客户端需要的接口，内部维护一个当前状态，并负责具体状态的切换。
> - 抽象状态（State）角色：定义一个接口，用以封装环境对象中的特定状态所对应的行为，可以有一个或多个行为。
> - 具体状态（Concrete State）角色：实现抽象状态所对应的行为，并且在需要的情况下进行状态切换。

根据状态模式的定义，小明将TaskState枚举类扩展成多个状态类，并具备完成状态的流转的能力；然后优化了任务类的实现：

```
// 任务状态抽象接口
interface State {
    // 默认实现，不做任何处理
    default void update(Task task, ActionType actionType) {
        // do nothing
    }
}
// 任务初始状态
class TaskInit implements State {
    @Override
    public void update(Task task, ActionType actionType) {
        if  (actionType == ActionType.START) {
            task.setState(new TaskOngoing());
        }
    }
}
// 任务进行状态
class TaskOngoing implements State {
    private ActivityService activityService;
    private TaskManager taskManager; 
    @Override
    public void update(Task task, ActionType actionType) {
        if (actionType == ActionType.ACHIEVE) {
            task.setState(new TaskFinished());
            // 通知
            activityService.notifyFinished(taskId);
            taskManager.release(taskId);
        } else if (actionType == ActionType.STOP) {
            task.setState(new TaskPaused());
        } else if (actionType == ActionType.EXPIRE) {
            task.setState(new TaskExpired());
        }
    }
}
// 任务暂停状态
class TaskPaused implements State {
    @Override
    public void update(Task task, ActionType actionType) {
        if (actionType == ActionType.START) {
            task.setState(new TaskOngoing());
        } else if (actionType == ActionType.EXPIRE) {
            task.setState(new TaskExpired());
        }
    }
}
// 任务完成状态
class TaskFinished implements State {

}
// 任务过期状态
class TaskExpired implements State {

}
@Data
class Task {
    private Long taskId;
    // 初始化为初始态
    private State state = new TaskInit();
    // 更新状态
    public void updateState(ActionType actionType) {
        state.update(this, actionType);
    }
}
```

小明欣喜地看到，经过状态模式处理后的任务类的耦合度得到降低，符合开闭原则。状态模式的优点在于符合单一职责原则，状态类职责明确，有利于程序的扩展。但是这样设计的代价是状态类的数目增加了，因此状态流转逻辑越复杂、需要处理的动作越多，越有利于状态模式的应用。除此之外，状态类的自身对于开闭原则的支持并没有足够好，如果状态流转逻辑变化频繁，那么可能要慎重使用。

处理完状态后，小明又根据老师的指导使用**观察者模式**去优化任务完成时的通知：

> 观察者模式：指多个对象间存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。这种模式有时又称作发布-订阅模式、模型-视图模式，它是对象行为型模式。观察者模式的主要角色如下。
>
> - 抽象主题（Subject）角色：也叫抽象目标类，它提供了一个用于保存观察者对象的聚集类和增加、删除观察者对象的方法，以及通知所有观察者的抽象方法。
> - 具体主题（Concrete Subject）角色：也叫具体目标类，它实现抽象目标中的通知方法，当具体主题的内部状态发生改变时，通知所有注册过的观察者对象。
> - 抽象观察者（Observer）角色：它是一个抽象类或接口，它包含了一个更新自己的抽象方法，当接到具体主题的更改通知时被调用。
> - 具体观察者（Concrete Observer）角色：实现抽象观察者中定义的抽象方法，以便在得到目标的更改通知时更新自身的状态。

小明首先设计好抽象目标和抽象观察者，然后将活动和任务管理器的接收通知功能定制成具体观察者：

```
// 抽象观察者
interface Observer {
    void response(Long taskId); // 反应
}
// 抽象目标
abstract class Subject {
    protected List<Observer> observers = new ArrayList<Observer>();
    // 增加观察者方法
    public void add(Observer observer) {
        observers.add(observer);
    }
    // 删除观察者方法
    public void remove(Observer observer) {
        observers.remove(observer);
    }
    // 通知观察者方法
    public void notifyObserver(Long taskId) {
        for (Observer observer : observers) {
            observer.response(taskId);
        }
    }
}
// 活动观察者
class ActivityObserver implements Observer {
    private ActivityService activityService;
    @Override
    public void response(Long taskId) {
        activityService.notifyFinished(taskId);
    }
}
// 任务管理观察者
class TaskManageObserver implements Observer {
    private TaskManager taskManager;
    @Override
    public void response(Long taskId) {
        taskManager.release(taskId);
    }
}
```

最后，小明将任务进行状态类优化成使用通用的通知方法，并在任务初始态执行状态流转时定义任务进行态所需的观察者：

```
// 任务进行状态
class TaskOngoing extends Subject implements State {  
    @Override
    public void update(Task task, ActionType actionType) {
        if (actionType == ActionType.ACHIEVE) {
            task.setState(new TaskFinished());
            // 通知
            notifyObserver(task.getTaskId());
        } else if (actionType == ActionType.STOP) {
            task.setState(new TaskPaused());
        } else if (actionType == ActionType.EXPIRE) {
            task.setState(new TaskExpired());
        }
    }
}
// 任务初始状态
class TaskInit implements State {
    @Override
    public void update(Task task, ActionType actionType) {
        if  (actionType == ActionType.START) {
            TaskOngoing taskOngoing = new TaskOngoing();
            taskOngoing.add(new ActivityObserver());
            taskOngoing.add(new TaskManageObserver());
            task.setState(taskOngoing);
        }
    }
}
```

最终，小明设计完成的结构类图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVm2AEU6AQbUAyCqHgKROft0Kw1MaCKWtIGT9nOia4JrfAicRdbLwXicjjnia8IA7YfY8PR2iabI1nbgAA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)任务模型设计_类图

通过观察者模式，小明让任务状态和通知方实现松耦合（实际上观察者模式还没能做到完全的解耦，如果要做进一步的解耦可以考虑学习并使用**发布-订阅模式**，这里也不再赘述）。

至此，小明成功使用状态模式设计出了高内聚、高扩展性、单一职责的任务的整个状态机实现，以及做到松耦合的、符合依赖倒置原则的任务状态变更通知方式。

“老师，我逐渐能意识到代码的设计缺陷，并学会利用较为复杂的设计模式做优化。”

“不错，再接再厉！”

## 4.活动的迭代重构

“小明，这次又有一个新的任务。”老师出现在正在认真阅读《设计模式》的小明的面前。

“好的。刚好我已经学习了设计模式的原理，终于可以派上用场了。”

“之前你设计开发了活动模型，现在我们需要在任务型活动的参与方法上增加一层风险控制。”

“OK。借此机会，我也想重构一下之前的设计。”

活动模型的特点在于其组成部分较多，小明原先的活动模型的构建方式是这样的：

```
// 抽象活动接口
interface ActivityInterface {
   void participate(Long userId);
}
// 活动类
class Activity implements ActivityInterface {
    private String type;
    private Long id;
    private String name;
    private Integer scene;
    private String material;
      
    public Activity(String type) {
        this.type = type;
        // id的构建部分依赖于活动的type
        if ("period".equals(type)) {
            id = 0L;
        }
    }
    public Activity(String type, Long id) {
        this.type = type;
        this.id = id;
    }
    public Activity(String type, Long id, Integer scene) {
        this.type = type;
        this.id = id;
        this.scene = scene;
    }
    public Activity(String type, String name, Integer scene, String material) {
        this.type = type;
        this.scene = scene;
        this.material = material;
        // name的构建完全依赖于活动的type
        if ("period".equals(type)) {
            this.id = 0L;
            this.name = "period" + name;
        } else {
            this.name = "normal" + name;
        }
    }
    // 参与活动
   @Override
    public void participate(Long userId) {
        // do nothing
    }
}
// 任务型活动
class TaskActivity extends Activity {
    private Task task;
    public TaskActivity(String type, String name, Integer scene, String material, Task task) {
        super(type, name, scene, material);
        this.task = task;
    }
    // 参与任务型活动
    @Override
    public void participate(Long userId) {
        // 更新任务状态为进行中
        task.getState().update(task, ActionType.START);
    }
}
```

经过自主分析，小明发现活动的构造不够合理，主要问题表现在：

1. 活动的构造组件较多，导致可以组合的构造函数太多，尤其是在模型增加字段时还需要去修改构造函数；
2. 部分组件的构造存在一定的顺序关系，但是当前的实现没有体现顺序，导致构造逻辑比较混乱，并且存在部分重复的代码。

发现问题后，小明回忆自己的学习成果，马上想到可以使用创建型模式中的**建造者模式**去做重构：

> 建造者模式：指将一个复杂对象的构造与它的表示分离，使同样的构建过程可以创建不同的表示。它是将一个复杂的对象分解为多个简单的对象，然后一步一步构建而成。它将变与不变相分离，即产品的组成部分是不变的，但每一部分是可以灵活选择的。建造者模式的主要角色如下:
>
> 1. 产品角色（Product）：它是包含多个组成部件的复杂对象，由具体建造者来创建其各个零部件。
> 2. 抽象建造者（Builder）：它是一个包含创建产品各个子部件的抽象方法的接口，通常还包含一个返回复杂产品的方法 getResult()。
> 3. 具体建造者(Concrete Builder）：实现 Builder 接口，完成复杂产品的各个部件的具体创建方法。
> 4. 指挥者（Director）：它调用建造者对象中的部件构造与装配方法完成复杂对象的创建，在指挥者中不涉及具体产品的信息。

根据建造者模式的定义，上述活动的每个字段都是一个产品。于是，小明可以通过在活动里面实现静态的建造者类来简易地实现：

```
// 活动类
class Activity implements ActivityInterface {
    protected String type;
    protected Long id;
    protected String name;
    protected Integer scene;
    protected String material;
    // 全参构造函数
   public Activity(String type, Long id, String name, Integer scene, String material) {
        this.type = type;
        this.id = id;
        this.name = name;
        this.scene = scene;
        this.material = material;
    }
    @Override
    public void participate(Long userId) {
        // do nothing
    }
    // 静态建造器类，使用奇异递归模板模式允许继承并返回继承建造器类
    public static class Builder<T extends Builder<T>> {
        protected String type;
        protected Long id;
        protected String name;
        protected Integer scene;
        protected String material;
        public T setType(String type) {
            this.type = type;
            return (T) this;
        }
        public T setId(Long id) {
            this.id = id;
            return (T) this;
        }
        public T setId() {
            if ("period".equals(this.type)) {
                this.id = 0L;
            }
            return (T) this;
        }
        public T setScene(Integer scene) {
            this.scene = scene;
            return (T) this;
        }
        public T setMaterial(String material) {
            this.material = material;
            return (T) this;
        }
        public T setName(String name) {
            if ("period".equals(this.type)) {
                this.name = "period" + name;
            } else {
                this.name = "normal" + name;
            }
            return (T) this;
        }
        public Activity build(){
            return new Activity(type, id, name, scene, material);
        }
    }
}
// 任务型活动
class TaskActivity extends Activity {
    protected Task task;
   // 全参构造函数
    public TaskActivity(String type, Long id, String name, Integer scene, String material, Task task) {
        super(type, id, name, scene, material);
        this.task = task;
    }
   // 参与任务型活动
    @Override
    public void participate(Long userId) {
        // 更新任务状态为进行中
        task.getState().update(task, ActionType.START);
    }
    // 继承建造器类
    public static class Builder extends Activity.Builder<Builder> {
        private Task task;
        public Builder setTask(Task task) {
            this.task = task;
            return this;
        }
        public TaskActivity build(){
            return new TaskActivity(type, id, name, scene, material, task);
        }
    }
}
```

小明发现，上面的建造器没有使用诸如抽象建造器类等完整的实现，但是基本是完成了活动各个组件的建造流程。使用建造器的模式下，可以先按顺序构建字段type，然后依次构建其他组件，最后使用build方法获取建造完成的活动。这种设计一方面封装性好，构建和表示分离；另一方面扩展性好，各个具体的建造者相互独立，有利于系统的解耦。可以说是一次比较有价值的重构。在实际的应用中，如果字段类型多，同时各个字段只需要简单的赋值，可以直接引用Lombok的@Builder注解来实现轻量的建造者。

重构完活动构建的设计后，小明开始对参加活动方法增加风控。最简单的方式肯定是直接修改目标方法：

```
public void participate(Long userId) {
    // 对目标用户做风险控制，失败则抛出异常
    Risk.doControl(userId);
    // 更新任务状态为进行中
    task.state.update(task, ActionType.START);
}
```

但是考虑到，最好能尽可能避免对旧方法的直接修改，同时为方法增加风控，也是一类比较常见的功能新增，可能会在多处使用。

“老师，风险控制会出现在多种活动的参与方法中吗？”

“有这个可能性。有的活动需要风险控制，有的不需要。风控像是在适当的时候对参与这个方法的装饰。”

“对了，装饰器模式！”

小明马上想到用**装饰器模式**来完成设计：

> 装饰器模式的定义：指在不改变现有对象结构的情况下，动态地给该对象增加一些职责（即增加其额外功能）的模式，它属于对象结构型模式。装饰器模式主要包含以下角色：
>
> 1. 抽象构件（Component）角色：定义一个抽象接口以规范准备接收附加责任的对象。
> 2. 具体构件（ConcreteComponent）角色：实现抽象构件，通过装饰角色为其添加一些职责。
> 3. 抽象装饰（Decorator）角色：继承抽象构件，并包含具体构件的实例，可以通过其子类扩展具体构件的功能。
> 4. 具体装饰（ConcreteDecorator）角色：实现抽象装饰的相关方法，并给具体构件对象添加附加的责任。

小明使用了装饰器模式后，新的代码就变成了这样：

```
// 抽象装饰角色
abstract class ActivityDecorator implements ActivityInterface {
    protected ActivityInterface activity;
    public ActivityDecorator(ActivityInterface activity) {
        this.activity = activity;
    }
    public abstract void participate(Long userId);
}
// 能够对活动做风险控制的包装类
class RiskControlDecorator extends ActivityDecorator {
    public RiskControlDecorator(ActivityInterface activity) {
        super(activity);
    }
    @Override
   public void participate(Long userId) {
        // 对目标用户做风险控制，失败则抛出异常
       Risk.doControl(userId);
        // 更新任务状态为进行中
        activity.participate(userId);
    }
}
```

最终，小明设计完成的结构类图如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/hEx03cFgUsVm2AEU6AQbUAyCqHgKROftuibJkJibU5BHr7pHeKicDI6ekm7TWN8eFrGDVpSFD8LLpzCHuibUzqn0UA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)活动迭代重构_类图

最终，小明通过自己的思考分析，结合学习的设计模式知识，完成了活动模型的重构和迭代。

“老师，我已经能做到自主分析功能特点，并合理应用设计模式去完成程序设计和代码重构了，实在太感谢您了。”

“设计模式作为一种软件设计的最佳实践，你已经很好地理解并应用于实践了，非常不错。但学海无涯，还需持续精进！”

## 5.结语

本文以三个实际场景为出发点，借助小明和老师两个虚拟的人物，试图以一种较为诙谐的“对话”方式来讲述设计模式的应用场景、优点和缺点。如果大家想要去系统性地了解设计模式，也可以通过市面上很多的教材进行学习，都介绍了经典的23种设计模式的结构和实现。不过，很多教材的内容即便配合了大量的示例，但有时也会让人感到费解，主要原因在于：一方面，很多案例比较脱离实际的应用场景；另一方面，部分设计模式显然更适用于大型复杂的结构设计，而当其应用到简单的场景时，仿佛让代码变得更加繁琐、冗余。因此，本文希望通过这种“对话+代码展示+结构类图”的方式，以一种更易懂的方式来介绍设计模式。

当然，本文只讲述了部分比较常见的设计模式，还有其他的设计模式，仍然需要同学们去研读经典著作，举一反三，学以致用。我们也希望通过学习设计模式能让更多的同学在系统设计能力上得到提升。

## 6.参考资料

- Gamma E. 设计模式: 可复用面向对象软件的基础 [M]. 机械工业出版社, 2007.
- 弗里曼. Head First 设计模式 [M]. 中国电力出版社, 2007.
- [oodesign.com](http://www.oodesign.com/)
- [java-design-patterns.com](http://java-design-patterns.com/)
- [**Java设计模式：23种设计模式全面解析**](http://c.biancheng.net/design_pattern/)

## 7.作者简介

嘉凯、杨柳，来自美团金融服务平台/联名卡研发团队。

原文链接：https://mp.weixin.qq.com/s/H2toewJKEwq1mXme_iMWkA