# The vision

- Billions of humanoid robots could be operating across the world by 2040, doing work well beyond the scope of today's factory robots.
- The robots look set to revolutionize sectors as varied as healthcare, public space maintenance, retail service and personal assistance.
- Companies and individuals alike can benefit from humanoid robots, but they will require clear guardrails to embrace this transformation.

from [WE forum](https://www.weforum.org/stories/2025/06/humanoid-robots-offer-disruption-and-promise/)

Robots that learn are robots that need what SUSE does best: a secure Linux base, containers,
fleet lifecycle management, and AI that runs privately, at the edge, under the owner's control.

# Where we are

Hackweek 2025 set an ambitious goal, a "SUSE-Enabled Robotics Stack" on SUSE Linux + K3s + ROS 2,
but only early exploration got done. This year the scope is narrower and hands-on: one real robot,
one complete pipeline, working end to end by Friday.

# Hackweek 2026 STEP

From a robot that runs on SUSE to a robot that **learns** on SUSE.

Build an end-to-end robot learning pipeline on a real SO-101 robot arm: the arm learns a task from human
demonstrations, performs it autonomously, and is orchestrated by a private AI agent. Everything runs locally
on open source software.

## Scope

- Teleoperate the arm, record demonstrations, train an imitation-learning policy (ACT) with Hugging Face LeRobot on a local GPU.
- Run the robot node on a Raspberry Pi 5 with openSUSE / SLE Micro and K3s; policy inference served from a GPU machine on the LAN.
- A local open-weights LLM (Ollama + Open WebUI, the building blocks of SUSE AI) drives the robot through an MCP server:
  "put the cube in the box and check you did it".
- Ship new policy versions to the robot fleet via GitOps with Fleet; live motor telemetry in Grafana.

## Deliverables

1. **Robot learning pipeline**
   - Scripts and runbook: teleop → dataset → training → deployment → evaluation, reproducible on SUSE.
2. **robotd + MCP server**
   - A small service that owns the arm (run task, stop, home, telemetry, camera snapshot) and exposes it to any AI agent via MCP.
3. **Edge deployment templates**
   - K3s manifests, Fleet GitOps setup, Grafana dashboard, plus a simulated robot to demo fleet-wide updates.
4. **Private AI agent demo**
   - Comparison of local open models vs a cloud model on the same robot-control scenario.
5. **Published results**
   - Open datasets and models on Hugging Face, evaluation reports, demo video.

## Outcome

The arm completes a pick-and-place task autonomously in at least 7 out of 10 attempts, orchestrated by an AI agent
with no cloud services involved — a concrete showcase of private AI at the edge on a SUSE stack.

## Follow along live 🎥

I'll stream the work all week: recording demonstrations, failed grasps, training results, the agent driving the arm.
Link: [TODO]

# Resources

- [slack channel #project-suse-robotics](https://suse.slack.com/archives/C0A09ULMPQX)
- [Hugging Face LeRobot](https://github.com/huggingface/lerobot)
- [SO-101 arm](https://github.com/TheRobotStudio/SO-ARM100)
- [K3s](https://k3s.io/) · [Fleet](https://fleet.rancher.io/)
- [Ollama](https://ollama.com/) · [Open WebUI](https://openwebui.com/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
