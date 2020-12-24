## ==Spring StateMachine==

### 引入依赖

创建一个Spring Boot的基础工程，并在`pom.xml`中加入`spring-statemachine-core`的依赖，具体如下：

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.3.7.RELEASE</version>
        <relativePath/> 
    </parent>
 
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.statemachine</groupId>
            <artifactId>spring-statemachine-core</artifactId>
            <version>1.2.0.RELEASE</version>
        </dependency>
    </dependencies>
```

### 创建枚举类

根据所述的订单需求场景定义状态和事件枚举，具体如下：

```java
//状态
public enum States {
    UNPAID,                 // 待支付
    WAITING_FOR_RECEIVE,    // 待收货
    DONE                    // 结束
}
 //状态迁移的事件
public enum Events {
    PAY,        // 支付
    RECEIVE     // 收货
}
```

其中共有三个状态（待支付、待收货、结束）以及两个引起状态迁移的事件（支付、收货），其中支付事件`PAY`会触发状态从待支付`UNPAID`状态到待收货`WAITING_FOR_RECEIVE`状态的迁移，而收货事件`RECEIVE`会触发状态从待收货`WAITING_FOR_RECEIVE`状态到结束`DONE`状态的迁移。

### 创建状态机配置类：

```java
import com.demo1.app.states.Events;
import com.demo1.app.states.States;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.EnableStateMachine;
import org.springframework.statemachine.config.EnumStateMachineConfigurerAdapter;
import org.springframework.statemachine.config.builders.StateMachineConfigurationConfigurer;
import org.springframework.statemachine.config.builders.StateMachineStateConfigurer;
import org.springframework.statemachine.config.builders.StateMachineTransitionConfigurer;
import org.springframework.statemachine.listener.StateMachineListener;
import org.springframework.statemachine.listener.StateMachineListenerAdapter;
import org.springframework.statemachine.transition.Transition;
 
import java.util.EnumSet;
 
/**
 * 创建状态机配置类
 * @EnableStateMachine 注解用来启用 Spring StateMachine状态机功能
 */
@Configuration
@EnableStateMachine
public class StateMachineConfig extends EnumStateMachineConfigurerAdapter<States, Events> {
 
    private Logger logger = LoggerFactory.getLogger(getClass());
 
    /**
     * configure(StateMachineStateConfigurer<States, Events> states)方法用来初始化当前状态机拥有哪些状态，
     * 其中initial(States.UNPAID)定义了初始状态为待支付UNPAID，
     * states(EnumSet.allOf(States.class))则指定了使用上一步中定义的所有状态作为该状态机的状态定义。
     * @param states
     * @throws Exception
     */
    @Override
    public void configure(StateMachineStateConfigurer<States, Events> states)
            throws Exception {
        states
            .withStates()
                .initial(States.UNPAID)                //初始状态 定义了初始状态为待支付UNPAID，
                .states(EnumSet.allOf(States.class));  //则指定了使用上一步中定义的所有状态作为该状态机的状态定义。
    }
 
    /**
     * configure(StateMachineTransitionConfigurer<States, Events> transitions)方法用来初始化当前状态机有哪些状态迁移动作，
     * 其中命名中我们很容易理解每一个迁移动作，都有来源状态source，目标状态target以及触发事件event。
     * @param transitions  StateMachineTransitionConfigurer<States, Events>
     * @throws Exception
     */
    @Override
    public void configure(StateMachineTransitionConfigurer<States, Events> transitions)
            throws Exception {
        transitions
                .withExternal()
                .source(States.UNPAID).target(States.WAITING_FOR_RECEIVE)// 指定状态来源和目标
                .event(Events.PAY)    // 指定触发事件
                .and()
                .withExternal()
                .source(States.WAITING_FOR_RECEIVE).target(States.DONE)
                .event(Events.RECEIVE);
    }
 
 
 
 
//     * configure(StateMachineConfigurationConfigurer<States, Events> config)方法为当前的状态机指定了状态监听器，
//     * 其中listener()则是调用了下一个内容创建的监听器实例，
//     * 用来处理各个各个发生的状态迁移事件。
//     * @param config
//     * @throws Exception
 
 
    @Override
    public void configure(StateMachineConfigurationConfigurer<States, Events> config)
            throws Exception {
        config
                .withConfiguration()
                .listener(listener());  // 指定状态机的处理监听器
    }
 
//
//     * StateMachineListener<States, Events> listener()方法用来创建StateMachineListener状态监听器的实例，
//     * 在该实例中会定义具体的状态迁移处理逻辑，上面的实现中只是做了一些输出，
//     * 实际业务场景会有更复杂的逻辑，所以通常情况下，
//     * 我们可以将该实例的定义放到独立的类定义中，并用注入的方式加载进来。
//     * @return
    @Bean
    public StateMachineListener<States, Events> listener() {
        return new StateMachineListenerAdapter<States, Events>() {
 
            @Override
            public void transition(Transition<States, Events> transition) {
                if(transition.getTarget().getId() == States.UNPAID) {
                    logger.info("订单创建，待支付");
                    return;
                }
                if(transition.getSource().getId() == States.UNPAID
                        && transition.getTarget().getId() == States.WAITING_FOR_RECEIVE) {
                    logger.info("用户完成支付，待收货");
                    return;
                }
 
                if(transition.getSource().getId() == States.WAITING_FOR_RECEIVE
                        && transition.getTarget().getId() == States.DONE) {
                    logger.info("用户已收货，订单完成");
                    return;
                }
            }
 
        };
    }

```

在该类中定义了较多配置内容，下面对这些内容一一说明：

- `@EnableStateMachine`注解用来启用Spring StateMachine状态机功能
- `configure(StateMachineStateConfigurer<States, Events> states)`方法用来初始化当前状态机拥有哪些状态，其中`initial(States.UNPAID)`定义了初始状态为`UNPAID`，`states(EnumSet.allOf(States.class))`则指定了使用上一步中定义的所有状态作为该状态机的状态定义。

```java
@Override
    public void configure(StateMachineStateConfigurer<States, Events> states)
            throws Exception {
        // 定义状态机中的状态
        states
            .withStates()
                .initial(States.UNPAID)    // 初始状态
                .states(EnumSet.allOf(States.class));
}
```

`configure(StateMachineTransitionConfigurer<States, Events> transitions)`方法用来初始化当前状态机有哪些状态迁移动作，其中命名中我们很容易理解每一个迁移动作，都有来源状态`source`，目标状态`target`以及触发事件`event`。

```java
 @Override
    public void configure(StateMachineTransitionConfigurer<States, Events> transitions)
            throws Exception {
        transitions
            .withExternal()
                .source(States.UNPAID).target(States.WAITING_FOR_RECEIVE)// 指定状态来源和目标
                .event(Events.PAY)    // 指定触发事件
                .and()
            .withExternal()
                .source(States.WAITING_FOR_RECEIVE).target(States.DONE)
                .event(Events.RECEIVE);
    }
```

`configure(StateMachineConfigurationConfigurer<States, Events> config)`方法为当前的状态机指定了状态监听器，其中 `listener()`则是调用了下一个内容创建的监听器实例，用来处理各个各个发生的状态迁移事件。

```java
   @Override
    public void configure(StateMachineConfigurationConfigurer<States, Events> config)
		throws Exception {
		config.withConfiguration()
			.listener(listener());    // 指定状态机的处理监听器
    }
```

`StateMachineListener<States, Events> listener()`方法用来创建 `StateMachineListener`状态监听器的实例，在该实例中会定义具体的状态迁移处理逻辑，上面的实现中只是做了一些输出，实际业务场景会会有更负责的逻辑，所以通常情况下，我们可以将该实例的定义放到独立的类定义中，并用注入的方式加载进来。

### 创建应用主类来完成整个流程：

```java

@SpringBootApplication
public class Application implements CommandLineRunner {
 
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
 
    @Autowired
    private StateMachine<States, Events> stateMachine;
 
    @Override
    public void run(String... args) throws Exception {
        stateMachine.start();
        stateMachine.sendEvent(Events.PAY);
        stateMachine.sendEvent(Events.RECEIVE);
    }
}
```

其中包括了状态监听器中对各个状态迁移做出的处理。

通过上面的例子，我们可以对如何使用Spring StateMachine做如下小结：

- 定义状态和事件枚举
- 为状态机定义使用的所有状态以及初始状态
- 为状态机定义状态的迁移动作
- 为状态机指定监听处理器

### 状态监听器

通过上面的入门示例以及最后的小结，我们可以看到使用Spring StateMachine来实现状态机的时候，代码逻辑变得非常简单并且具有层次化。整个状态的调度逻辑主要依靠配置方式的定义，而所有的业务逻辑操作都被定义在了状态监听器中，其实状态监听器可以实现的功能远不止上面我们所述的内容，它还有更多的事件捕获，我们可以通过查看`StateMachineListener`接口来了解它所有的事件定义：

```java

public interface StateMachineListener<S,E> {
 
    void stateChanged(State<S,E> from, State<S,E> to);
 
    void stateEntered(State<S,E> state);
 
    void stateExited(State<S,E> state);
 
    void eventNotAccepted(Message<E> event);
 
    void transition(Transition<S, E> transition);
 
    void transitionStarted(Transition<S, E> transition);
 
    void transitionEnded(Transition<S, E> transition);
 
    void stateMachineStarted(StateMachine<S, E> stateMachine);
 
    void stateMachineStopped(StateMachine<S, E> stateMachine);
 
    void stateMachineError(StateMachine<S, E> stateMachine, Exception exception);
 
    void extendedStateChanged(Object key, Object value);
 
    void stateContext(StateContext<S, E> stateContext);
 
}
```

### 注解监听器

对于状态监听器，Spring StateMachine还提供了优雅的注解配置实现方式，所有`StateMachineListener`接口中定义的事件都能通过注解的方式来进行配置实现。比如，我们可以将之前实现的状态监听器用注解配置来做进一步的简化：

```java

@WithStateMachine
public class EventConfig {
 
    private Logger logger = LoggerFactory.getLogger(getClass());
 
    @OnTransition(target = "UNPAID")
    public void create() {
        logger.info("订单创建，待支付");
    }
 
    @OnTransition(source = "UNPAID", target = "WAITING_FOR_RECEIVE")
    public void pay() {
        logger.info("用户完成支付，待收货");
    }
 
    @OnTransition(source = "WAITING_FOR_RECEIVE", target = "DONE")
    public void receive() {
        logger.info("用户已收货，订单完成");
    }
 
}
```

上述代码实现了与快速入门中定义的`listener()`方法创建的监听器相同的功能，但是由于通过注解的方式配置，省去了原来事件监听器中各种if的判断，使得代码显得更为简洁，拥有了更好的可读性。