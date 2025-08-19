# Flxbl - Continuous Delivery

### Links

| Website       | [https://www.fxlbl.io/](https://www.fxlbl.io/)               |
| ------------- | ------------------------------------------------------------ |
| Documentation | [https://docs.flxbl.io/flxbl](https://docs.flxbl.io/flxbl)   |
| Codebase      | [https://github.com/flxbl-io/](https://github.com/flxbl-io/) |

### Use case

flxbl addresses the painful reality of enterprise Salesforce DevOps:

**1. Deployment Hell**  
Traditional Salesforce deployments are a nightmare. Hours-long deployments, manual cherry-picking between release branches, deployment failures at the last minute, and that one person who knows the "right" order to deploy things. Without enforced structure, deployments become a full-time job for someone on the team.

**2. Forced Modularity**  
Like fflib forces code structure, flxbl forces deployment structure. It won't work unless you properly modularize your org. No more giant, monolithic deployments where everything depends on everything else. You're forced to think in packages, boundaries, and clean dependencies from day one.

**3. Shift-Left Testing**  
Instead of finding deployment issues after merging (requiring painful rollbacks), flxbl validates everything during the PR phase. Every PR gets its own environment, automatically validated before merge. When deployment fails, you fix the PR - not roll back commits from main.

**4. Single Trunk Development**  
No more release branch hell. No more cherry-picking commits. No more "which branch has the latest version?" confusion. Single trunk with versioned packages means you can release fast and often. Deploy what's ready, when it's ready.

**5. Incremental Deployments**  
Deploy individual packages, not the entire org. A fix to the revenue module doesn't require redeploying your entire lead management system. This granularity reduces risk and deployment time from hours to minutes.

### Rationale

**Why flxbl over point-and-click tools:**

**The Enforced Process Advantage**  
Gearset, Copado, and similar tools are powerful but they're ultimately just tools. They'll happily deploy your mess faster. flxbl is opinionated - it enforces modularity or it simply won't work. This constraint is its strength.

**Modularity as a Requirement**  
You can't half-implement flxbl. Your code must be properly modularized into packages with clear dependencies. This forced structure pays dividends:
- Parallel development without conflicts
- Independent testing and deployment
- Clear ownership boundaries
- Genuine reusability

**Developer-First Approach**  
While point-and-click tools add UI layers, flxbl integrates into developer workflows:
- Git is the source of truth
- CI/CD pipelines as code
- Version control for everything

**The Journey to Modularity**

**For Existing Orgs:**  
Moving to modularity is a journey, not a sprint. We advocate incremental modularity:
1. Start with new features as packages
2. Extract stable, low-change components first
3. Gradually refactor high-value areas
4. Accept that some legacy code may never be packaged

Trying to modularize everything at once is a recipe for failure. Pick your battles.

**For New Implementations:**  
Start modular from day one. It's essential to:
- Keep packages small and focused
- Follow the modularity guidelines strictly
- Resist the temptation to create "misc" or "utils" packages
- Design clear interfaces between packages

**The Real Cost-Benefit:**

**Benefits:**
- Deployments go from hours to minutes
- No more deployment-day surprises
- Developers can work independently
- Rollbacks are package-specific, not org-wide
- True continuous delivery becomes possible

**Costs:**
- High initial learning curve for modularity
- Requires strong architectural discipline
- Package dependency management adds complexity
- Refactoring existing orgs is a significant investment

**Our Verdict:**  
For teams committed to continuous delivery and willing to invest in proper architecture, flxbl is transformative. For teams wanting a quick fix to deployment problems without changing how they build, look elsewhere.

### Alternatives

| Tool/Approach | When It Works | When It Doesn't | Our Take |
| --- | --- | --- | --- |
| **Gearset** | Want UI-based deployments, comparison tools, rollback capabilities | Need enforced modularity, want git-native workflows | Great tool but doesn't enforce good architecture. Can deploy bad code faster |
| **Copado** | Enterprise with budget, want full DevOps suite, compliance requirements | Small teams, limited budget, developer-first culture | Comprehensive but expensive. Can become a crutch that hides architectural problems |
| **Salesforce CLI + Scripts** | Small team, simple requirements, high technical capability | Complex dependencies, multiple developers, need orchestration | Works until it doesn't. Scripts become unmaintainable as complexity grows |
| **SFDX + GitHub Actions** | Want free, simple CI/CD, comfortable with YAML | Need package management, complex orchestration, multi-org deployments | Good starting point but lacks advanced features. Consider migrating to flxbl as you grow |
| **Salto** | Want declarative change tracking, business user friendly | Need package-based architecture, complex dependencies | Focuses on different problem - configuration management vs architectural modularity |
| **Manual Change Sets** | Very simple requirements, infrequent deployments | Everything else | Please don't. It's 2024 |

### Implementation Guidance

**Critical Success Factors:**

1. **Understand Modularity First**  
   Before touching flxbl, understand modular architecture. Read our modularity guidelines and anti-patterns documentation. flxbl is just a tool - modularity is the discipline.

2. **Start Small with New Features**  
   Don't try to modularize your entire org at once. Start with:
   - New features as independent packages
   - Extracted utility functions
   - Stable, low-change components
   Each success builds confidence and knowledge.

3. **Package Boundaries Are Sacred**  
   Once you define a package boundary, defend it fiercely. No "temporary" cross-package dependencies. No "we'll fix it later" violations. The moment you compromise, you're back to monolithic chaos.

4. **Follow the Branching Model**  
   Stick to the flxbl-recommended branching strategy (documented in their guides). Don't try to adapt it to your old branching model - you'll lose the benefits.

5. **Incremental Migration Strategy**  
   For existing orgs:
   - Phase 1: New features in packages
   - Phase 2: Extract stable utilities
   - Phase 3: Modularize high-value domains
   - Phase 4: Tackle legacy code (if worth it)
   
   Many orgs successfully run hybrid - some code packaged, some not. That's fine.

6. **Package Design Principles**  
   - **Small and Focused**: One package, one purpose
   - **Clear Interfaces**: Public APIs, everything else private
   - **Minimal Dependencies**: Fewer dependencies = easier deployment
   - **Version Everything**: Semantic versioning from day one

7. **Common Pitfalls:**  
   - Creating "common" or "utils" packages that become dumping grounds
   - Circular dependencies between packages
   - Too many small packages (maintenance overhead)
   - Too few large packages (loses modularity benefits)
   - Ignoring the CI/CD pipeline until "later"

8. **Investment Required:**  
   - 2-3 months to properly modularize a medium-complexity org
   - Ongoing architectural governance
   - Team training on package-based development
   - CI/CD pipeline setup and maintenance

**When Not to Use flxbl:**  
- Simple orgs with infrequent changes
- Teams without architectural discipline
- Organizations unwilling to invest in modularity
- Quick prototypes or POCs

The key message: flxbl enables continuous delivery, but only if you commit to modular architecture. Half-measures will frustrate everyone and deliver no value.
