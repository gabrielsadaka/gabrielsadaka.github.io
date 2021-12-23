# Build software assuming you will get it wrong every step of the way

Modern software systems are continuously evolving to ever growing levels of complexity. The advancements in the ecosystem that have enabled organisations of all sizes to deliver innovative experiences to their customers, has at the same time made it virtually impossible for any individual to understand the entire system. As cloud providers continue to deliver new services at a rapid pace, frontend web and mobile experiences continue to advance and emerging technologies disrupt the industry, the complexity that underpins modern software systems continues to grow.

As customers see what is possible with modern technology, they will move from being impressed by it, to expecting it from every system they interact with. Increasing the complexity of software systems, requiring organisations to grow, so that they are able to deliver and operate them, continuously increasing the likelihood of something going wrong.

There are two polarising approaches to designing systems and processes that deal with the complexity:

1. Risk averse - optimising for the hope that one will make the right decision at every point in the journey.
2. Embrace risk - optimising for the assumption that it is likely that one will make the wrong decision at every point in the journey.

**Approach 1 - Risk Averse**

The first approach, where the hope is that one will always make the right decision, tends to be common in traditional enterprises, where practises and structures exist to prevent mistakes from occurring. They are commonly represented by change review boards, manual release gates, centralised decision making and extensive release checklists. While these practises and structures exist with the best of intentions, to try and ensure that the delivery teams do not introduce changes that could lead to incidents, they instead result in slowing down the organisation’s ability to react to change and respond to failure swiftly.

Assuming every decision is right, results in big upfront software architecture and long term detailed planning. Although this sounds similar to the waterfall software delivery methodology, many organisations that aim to follow agile principles, happen to also follow practises that are not adaptable to changing requirements and assume that all of the big decisions can be made upfront. This inevitably results in longer lead times and time to recovery, due to the inability to move fast and respond to rapidly changing circumstances.

**Approach 2 - Embrace Risk**

The alternative approach to managing the ever growing complexity of software systems, is to assume that it's likely for one to make the wrong decision every step of the way. Although this may sound like chaos, it doesn’t have to be that way, software can be delivered safely and efficiently, even when failure is a common occurrence in the delivery lifecycle. To ensure the software is delivered safely, automated checks and balances can be introduced at every stage, to validate it meets the expectations of the organisation and its consumers.

The only secure, 100% fault tolerant system is one that exists in a vacuum, not connected to any network and that cannot be interacted with by a human. Every other system can and will fail at some point. Systems that are resilient can evolve to meet the continuous evolution of customer demand, by building safety into every step, to protect from the failures that will occur.

Every assumption, every hypothesis we test, every decision made has the possibility of resulting in failure. From determining what is built to meet the customer’s needs to operating the software in production, the journey is fraught with risk and it is critical that we embrace risk rather than avert it. Instead of aiming to build systems that can never fail, build systems that are resilient to failure.

Modern software delivery practises can be embedded in every part of the journey to enable the delivery of innovative experiences that embrace risk, in a safe and efficient manner. Thus ensuring that the systems will support the organisation to respond to failure, providing the necessary automated safeguards to prevent catastrophes. It enables the organisation to learn from the failure and improve the system to reduce the likelihood of it occurring again.

_The following is a non-exhaustive list of techniques and practises that can help embrace risk safely, all of which are designed to deal with the reality that things will go wrong._

The following techniques and practises can be used to understand the risks and manage them on a regular basis:

- **Agile threat modelling**- regularly identify the security risks in order to design and develop mitigations to them.
- **Incident and disaster recovery plans** - prepare for the inevitable system failure by developing plans on how to respond to them.
- **Blameless post incident reviews** - learn from system failure, developing clear steps on how to prevent it from occurring again, recognising that the failure is due to a flaw in the system not the individuals involved.
- **Risk management** - map out an understanding of all key risks and their relative impact/likelihood, to develop potential mitigation plans, reducing the likelihood that they will occur and their blast radius.

Once the risks are understood, it is important to embed an understanding of them throughout the entire lifecycle, embedding quality and verification into every step of the software development lifecycle.

- **Understand the customer needs** - iteratively understand the customer problem, designing potential solutions, using agile principles to gain rapid feedback.
- **Design the solution** - go from big upfront architecture, assuming that enough is known upfront to make the big decisions, to evolutionary architecture and domain driven design, building systems that naturally and safely evolve over time, as you learn more, to meet the ever changing needs of the organisation.
- **Develop it** - automate testing to validate software works from the beginning and that future changes don't break existing code. Adopt fitness functions, static analysis and code coverage, at build time to validate the software is evolvable and maintainable.
- **Deploy it** - automatically deploy the software and infrastructure to make deployment a risk free activity. Automate compliance checks, security/secret scanning and dependency updating, to ensure that the deployed software is secure and verified.
- **Operate it** - prepare for the unexpected, expecting that incidents will occur, by automating the understanding of how the system operates in production, using modern observability and alerting tooling. Understand the signals of failure and continuously test the ability to respond to them using resilience engineering.
- **Keep the data safe & consistent** - continuously backup all of the data the system produces and collects, regularly testing recovery plans to ensure they are ready when needed. Ensure the data is protected by encrypting it in transit and at rest.
- **Protect the system** - adopt defence in depth techniques, ensuring that every aspect of the system is protected, assuming that attackers can penetrate any part of it.
- **Learn** - continuously measure and evaluate the system to understand how customers are using it and whether it is meeting expectations, feeding that back into every step of the process to improve it.

Enable your development teams to move fast and deliver software safely & securely by embedding verification, compliance & security into every step of the lifecycle. Give them the freedom and the confidence to develop and deploy code, trusting that the systems that exist will protect them when mistakes are made or a hypothesis is proven to be false. Rather than adopting risk averse approaches, embrace risk and accept that it is part of delivering software in an ever growing complex environment, encouraging the organisation to take risks & innovate.

It is important to consider that not all risks are necessarily worth taking though. The reward that could be potentially gained from the risk needs to be considered to ensure that it is worth the potential loss if things go wrong.

As the seminal book, _“Accelerate: The Science of Lean Software and DevOps: Building and Scaling High Performing Technology Organisations”_, has demonstrated, organisations with high performing engineering teams are twice as likely to exceed their organisational goals. Moving fast and _not_ breaking things, but assuming things can and will break, then factoring that into how we build and deliver software will enable organisations to embrace risk and deliver rapid innovation safely. Slowing down how we deliver software, not only limits our ability to innovate, but can also limit our ability to respond to incidents and defects. Systems that enable rapid delivery of software can save us when things go wrong, as they definitely will go wrong.

When designing software delivery processes, ask at every step of the way, how can something go wrong and how can it be prevented or how can the impact of failure be reduced. Don’t design departments of no, design systems of yes, but do so safely, enabling the organisation to embrace the risk, not avert it.
