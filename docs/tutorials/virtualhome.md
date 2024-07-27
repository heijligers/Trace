# 🤯 Finally: Emergent Behaviors

In the previous two sections, we showed how to decorate Python functions for Trace to optimize and how to create an RL agent using Trace. 
Now, we will demonstrate how Trace can be used to create interactive agents that learn emergent behaviors in a multi-agent environment.

```python
import opto.trace as trace
from opto.trace import node, bundle, model, GRAPH
from opto.optimizers import OptoPrime
```

## 🏠 VirtualHome

[VirtualHome](http://virtual-home.org/) is a Unity engine based simulation environment that creates a home-like enviornment where multiple agents need to collaboratively solve a 
series of tasks, ranging from book reading, putting empty plates in a dishwasher, to preparing food.

```{image} ../images/virtualhome_image.png
:alt: virtual-home
:align: center
```
(Image credit: [Virtual Home Project Page](http://virtual-home.org/))

LLM agents have been observed to have "natural" organizational behavior in LLM-based agents when they were asked to have a few rounds of discussion, before carrying out the tasks in the environment ([Prior work](https://organized-llm-agents.netlify.app/)).
However, it's hard to determine whether the behavior of LLM agents is a result of emergence or if it's due to inherent biases towards hierarchical organizational
behaviors because they are trained on human data.

In this section, we show that how to create multiple agents using Trace, and through optimization, we can observe emergent behaviors in these agents.
We first construct an agent.

```{tip}
We can define a class that inherits from multiple other classes and still use `@model` to decorate it and capture its behavior.
```

The observation of the agent and the action they can take in virtual home can be expressed in text. We follow the convention from [Guo et al. (2024)](https://arxiv.org/abs/2403.12482)
with minor modifications and here is an example:

```text
I'm Agent_2. I'm in a hurry to finish the housework with my friends Agent_1 together. 
Given our shared goal, dialogue history, and my progress and previous actions, 
please help me choose the best available action to achieve the goal as soon as possible. 
Note that I can hold two objects at a time and there are no costs for holding objects. 
All objects are denoted as <name> (id), such as <table> (712). Be aware that exploring 
or checking the items in another room may cost some time to walk there.

Goal: Find and put 1 apple, 1 wine, 1 pudding onto the <coffeetable> (268).

Progress: This is step 0. I'm holding nothing. I'm in the bedroom, where I found an unchecked container <cabinet> (216). I don't know where Agent_1 is. The livingroom is unexplored. The kitchen is unexplored. The bathroom is unexplored. 

Dialogue history:

Previous actions: [goexplore] <bedroom> (210)
Available actions:
A. [goexplore] <livingroom> (267)
B. [goexplore] <kitchen> (11)
C. [goexplore] <bathroom> (172)
D. [gocheck] <cabinet> (216)

Response format:
{"thoughts" : "thoughts content", 
"action" : "choose one action from the available actions above"}

Note: You must respond in the json format above. The action choice must be the same as one of the available actions. 
If there's nothing left to do, the action can be "None". If you choose [send_message], you must also generate the actual message.
```

For the Trace optimzied agent, we additionally add `Plan:$PLAN` right below `Goals`. The agent stores a plan in its python class object (which serves as its **memory**),
and when it needs to produce an action, it will replace `$PLAN$` with the current plan.
Trace optimizer will update the **plan** based on the feedback from the environment and the current progress.

```python
import json
import random
from examples.virtualhome import LLMCallable, BaseUtil

@model
class Agent(LLMCallable, BaseUtil):
    def __init__(self, verbose=False):
        super().__init__(verbose=verbose)
        self.plan = node("", trainable=True, 
                         description="This represents the current plan of the agent.")

    def __call__(self, obs):
        obs = obs.replace("$PLAN$", self.plan)
        action = self.act(obs)
        return action

    @bundle()
    def act(self, obs):
        """
        Call the LLM to produce the next action for the agent
        """
        response = self.call_llm(obs)
        available_actions = self.extract_actions(obs)
        plan = json.loads(response)
        if 'action' in plan:
            action = plan['action']
        else:
            action = ""

        if '[send_message]' not in action:
            action = self.unify_and_match_action(action, available_actions)

            # if the matched failed, we randomly choose an action.
            if action not in available_actions.values():
                action = random.choice(list(available_actions.values()))

        return action
```

In a multi-agent environment, we can create multiple agents and let them interact with each other.
We take a synchronous approach, where all agents take actions after observing the current state of the environment, and their
actions are executed together. To make the simulation faster, we implement a sticky-action mechanism, where if the environment
observation is the same as the previous observation, we repeat the previous action without making another LLM call.

```{note}
The full virtualhome environment requires Unity engine executable and is not included in the Trace package.
This code is for demo purposes only.
```

We first create two agents and their corresponding optimizers.
```python

agent1 = Agent()
agent2 = Agent()

optimizer1 = OptoPrime([agent1.plan])
optimizer2 = OptoPrime([agent2.plan])

agents = [agent1, agent2]
```

We then run the simulation for a fixed number of steps. In each step, we observe the environment, and each agent produces an action based on its observation.

```python
from examples.virtualhome import VirtualHomeEnv

horizon = 50

env = VirtualHomeEnv()

# we specify a task in this environment
agent_obs, agent_obs_descs, agent_goal_specs, agent_goal_descs, agent_infos = env.reset(task_id=8)

for h in range(horizon):
    plans, errors = {}, {}
    for i in range(len(agents)):
        agent = agents[i]
        try:
            plans[i] = agent(agent_obs_descs[i])
        except trace.ExecutionError as e:
            errors[i] = e
            plans[i] = None
            break
    
    if len(errors) == 0:
        step_info, next_agent_obs_descs, dict_actions, dict_info = env.step(plans)
        _, reward, done, infos, messages = step_info
    
    for i in range(len(agents)):
        optimizer = optimizers[i]
        optimizer.zero_feedback()
        feedback = f"Task Return: {sum(reward[i]['reward'])}"

        print(f"Step: {h}")
        print(f"Feedback: {feedback}")

        optimizer.backward(next_agent_obs_descs[i], feedback)
        optimizer.step(verbose=False)
        
        # now we detach the graph for the next step
        agent_obs_descs[i] = next_agent_obs_descs[i].detach()
```

```{note}
Here we see an interesting case of optimization. In this environment, the observation an agent sees is the result of the agent's previous action.
Therefore, we can directly call `backward` on the next observation.
```

```{tip}
To learn more about how to use Trace to create an agent in an interactive environment, check out the [Meta-World](https://microsoft.github.io/Trace/examples/code/metaworld.html) example.
```

We can see the evolution of the agents' plans and actions over time. The agents will learn to coordinate with each other to achieve the shared goal.
Here, we show some examples of the agent's plans and their actions at different steps.

```text
Plan: To optimize our efforts, Agent_1 can grab <plate> (374) as it is with her in the livingroom. Meanwhile, Agent_2, already nearby, could quickly explore the unexplored bathroom for any hidden items that might be of use, ensuring efficient use of time and effort. Maintaining communication will ensure that if anything of immediate use is found in the bathroom, it can be prioritized. This will prevent any time wastage in back-and-forth movements.
```

We compare with the baseline ReAct agents that only outputs `thoughts` before taking an action, but without the optimizer changing the plan at each step.

```{image} ../images/TRACE_fig-6.png
:alt: task-reward
:align: center
```

Continue...