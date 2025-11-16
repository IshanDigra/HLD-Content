# Horizontal vs Vertical Scaling

## Overview
- **Definition:** Scaling changes system capacity for more load.
- **Horizontal (Scale Out):** Add more machines/nodes.
- **Vertical (Scale Up):** Add more CPU, RAM, disk to a machine.
- **Prerequisite:** Know basics of distributed systems and cloud.
- **Difficulty:** Beginner

## Key Concepts
| Concept       | Description                        |
|--------------|------------------------------------|
| Horizontal   | More nodes for more load           |
| Vertical     | Bigger/stronger single machine     |
| Diagonal     | Scale up, then scale out if needed |

## Visual Summary
```
Horizontal:  [Node] [Node] [Node] [+ more]
Vertical:    [Node++ (more RAM/CPU)]
```

## Learning Path
| Step     | Link                                                                  |
|----------|-----------------------------------------------------------------------|
| Article  | https://www.cloudzero.com/blog/horizontal-vs-vertical-scaling/        |

## When to Use
| Use For           | Reason                                                               |
|-------------------|----------------------------------------------------------------------|
| Horizontal        | Handle rapid growth, fault tolerance, distributed/users in regions   |
| Vertical          | Cost-effective, simple system, can tune up existing server specs     |
| Diagonal          | Start vertical, move to horizontal for big systems                   |

## Trade-offs
| Pros (Horizontal)    | Cons (Horizontal)       |
|----------------------|------------------------|
| Resilient, scalable  | More complex, costlier |
| No single point fail | Needs load balancer    |
| Pros (Vertical)      | Cons (Vertical)        |
|----------------------|------------------------|
| Simple, cheap start  | Failure=fatal, limits  |
| One box to manage    | Scaling hits a ceiling |

## Examples
- Google/AWS: Auto-scale with more servers
- Add RAM/CPU to a DB server for fast upgrade

## Interview Tips
- Q: How to handle 10x user spike? A: Start with vertical, then plan for horizontal

## Quick Reference
- Horizontal: Add nodes. Vertical: Add resources.

## Revision Checklist
- [ ] Know the difference + trade-offs
- [ ] Draw both diagrams
- [ ] Know when to use each
- [ ] See article above

**Last Updated:** Nov 2025
