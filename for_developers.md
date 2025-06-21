# MetaGPT for Developers

This document provides developers with an understanding of the MetaGPT framework, its architecture, and how to work with the codebase.

## 1. Overview

MetaGPT is a multi-agent framework that leverages Large Language Models (LLMs) to simulate a software development company. It can take a high-level user requirement (an "idea") and autonomously generate various project artifacts, including Product Requirement Documents (PRDs), system designs, task lists, and source code.

The core philosophy behind MetaGPT is **"Code = SOP(Team)"**, where SOP stands for Standard Operating Procedures. This means the collaboration between AI agents (Roles) within a Team is structured and follows predefined procedures to accomplish complex tasks.

## 2. Architecture

MetaGPT's architecture is built around several key components:

*   **`SoftwareCompany` (`metagpt/software_company.py`):**
    *   The main entry point for running the MetaGPT simulation, typically via the CLI.
    *   It initializes a `Team` with a standard set of `Role`s and provides the initial user `idea`.

*   **`Team` (`metagpt/team.py`):**
    *   Manages a collection of `Role`s (agents).
    *   Contains an `Environment` for communication and shared state.
    *   Orchestrates the overall project workflow by running multiple rounds of agent interactions.
    *   Manages the project's budget (`investment`) and can be serialized/deserialized for saving and resuming projects.

*   **`Role` (Base: `metagpt/roles/role.py`, Implementations: `metagpt/roles/*`):**
    *   Represents an AI agent with a specific profile (e.g., Product Manager, Architect, Engineer), goals, and constraints.
    *   Key methods:
        *   `_observe()`: Monitors the `Environment` for relevant messages.
        *   `_think()`: Decides on the next `Action` to take based on observations and internal state.
        *   `_act()`: Executes the chosen `Action`.
        *   `_react()`: The main operational loop for a role.
    *   Roles communicate by publishing `Message` objects to the `Environment`.
    *   Some advanced roles are based on `RoleZero` (`metagpt/roles/di/role_zero.py`), which provides enhanced capabilities for tool usage and experience retrieval.

*   **`Action` (Base: `metagpt/actions/action.py`, Implementations: `metagpt/actions/*`):**
    *   Represents a specific, atomic task that a `Role` can perform (e.g., `WritePRD`, `WriteDesign`, `WriteCode`).
    *   The `run()` method executes the task, often involving interaction with an LLM using specific prompts to generate content or make decisions.
    *   Actions typically read from and write to the `ProjectRepo`, producing artifacts like documents or code files.
    *   `ActionNode` (`metagpt/actions/action_node.py`) is often used to structure LLM calls with predefined prompt templates and parsing logic.

*   **`Environment` (`metagpt/environment/environment.py`, Base: `metagpt/environment/base_env.py`):**
    *   Serves as the communication backbone and shared context for `Role`s within a `Team`.
    *   Manages a message queue (`memory`) where `Role`s publish and subscribe to `Message`s.
    *   Tracks project history and manages the execution flow of agent interactions in rounds.

*   **`Message` (`metagpt/schema.py`):**
    *   The primary data structure for communication between `Role`s via the `Environment`.
    *   Contains content, sender, recipient, and the `Action` that caused the message.
    *   Can also include `instruct_content` for structured data payload.

*   **`ProjectRepo` (`metagpt/utils/project_repo.py`):**
    *   A utility for managing the file structure of the generated software project.
    *   It organizes different types of artifacts (requirements, PRDs, system designs, source code, test outputs, etc.) into distinct directories using `FileRepository` objects.

*   **LLM Providers (`metagpt/provider/*`):**
    *   Abstracts the interaction with various LLM APIs (e.g., OpenAI, Azure, Anthropic). Configuration is handled via `config2.yaml`.

*   **Prompts (`metagpt/prompts/*`):**
    *   Contains templates and instructions used by `Action`s to guide LLM behavior for specific tasks.

## 3. Typical Workflow

The typical workflow in MetaGPT for generating a software project is as follows:

1.  **Initialization:**
    *   A user provides an `idea` (e.g., "Create a 2048 game") via the CLI (`metagpt "idea"`) or by calling `generate_repo()`.
    *   `SoftwareCompany` sets up a `Team` with predefined roles (ProductManager, Architect, Engineer, etc.) and an `Environment`.
    *   The initial `idea` is published as a `Message` in the `Environment`.

2.  **Product Definition (ProductManager):**
    *   The `ProductManager` role observes the `idea`.
    *   It executes the `WritePRD` action, which interacts with an LLM using relevant prompts.
    *   A Product Requirement Document (PRD) is generated and saved to the `ProjectRepo`.

3.  **System Design (Architect):**
    *   The `Architect` role observes the generated PRD.
    *   It executes the `WriteDesign` action.
    *   System design documents, including API specifications, data models, and sequence diagrams (often as Mermaid charts), are created and saved.

4.  **Task Planning (ProjectManager - assumed):**
    *   (While not explicitly detailed in the initial file read, a `ProjectManager` role typically exists).
    *   The `ProjectManager` would observe the PRD and system design.
    *   It would execute an action like `WriteTasks` to break down the development work into smaller, manageable coding tasks. These tasks are saved.

5.  **Code Implementation (Engineer):**
    *   The `Engineer` role observes the design documents and coding tasks.
    *   It executes the `WriteCode` action for each task.
        *   If incremental mode (`config.inc`) is enabled, it might first run `WriteCodePlanAndChange` to outline changes before coding.
        *   The `WriteCode` action uses prompts that include the design, task description, and potentially existing code context.
        *   Generated source code is saved to the `ProjectRepo`.
    *   After coding, the `Engineer` might run `SummarizeCode` to get an LLM-based assessment of the generated code, potentially leading to revisions.
    *   If code review is enabled, a `WriteCodeReview` action might be involved.

6.  **Testing and Bug Fixing (QaEngineer - assumed):**
    *   (Typically, a `QaEngineer` role would exist).
    *   The `QaEngineer` would observe completed code.
    *   It would execute `WriteTest` to generate unit tests, then `RunCode` to execute them.
    *   If tests fail or bugs are reported, the `FixBug` action (often handled by an `Engineer`) would be triggered.

7.  **Iteration and Completion:**
    *   This cycle of roles observing messages/artifacts, thinking, and acting continues in rounds.
    *   The `Team` coordinates these rounds until the project goals are met, the allocated `n_round` limit is reached, or the `investment` (budget) is depleted.
    *   The final output is a project structure in the `workspace/` directory containing all generated artifacts.

## 4. Key Directories and Files

*   **`metagpt/`**: Root directory for the core application logic.
    *   **`actions/`**: Contains implementations of specific `Action` classes (e.g., `write_prd.py`, `write_code.py`). These define the atomic operations agents can perform.
    *   **`roles/`**: Defines the different agent archetypes (`Role` subclasses like `product_manager.py`, `engineer.py`).
    *   **`prompts/`**: Stores prompt templates and instructions used to guide LLM interactions for various actions and roles.
    *   **`tools/`**: Implements utilities and integrations that agents can use (e.g., web browsers, search engines, code execution shells).
    *   **`provider/`**: Handles communication with different LLM service providers (OpenAI, Azure, etc.).
    *   **`environment/`**: Manages agent communication, message passing, and the overall state of a `Team`'s collaborative effort.
    *   **`memory/`**: Components related to agent memory, enabling them to recall past interactions and context.
    *   **`schema.py`**: Defines core data structures like `Message`, `Document`, and various context objects used throughout the system.
    *   **`configs/` & `config2.py`**: Manages system configuration, including LLM settings, workspace paths, and feature flags.
    *   **`llm.py`**: Centralizes LLM interaction logic.
    *   **`software_company.py`**: Orchestrates the creation of a `Team` and the overall software generation process from a user's idea.
    *   **`team.py`**: Defines the `Team` class, which groups and manages `Role`s.
    *   **`utils/`**: Contains common utility functions and classes.
        *   `project_repo.py`: Manages the file system structure for generated projects.
        *   `file_repository.py`: Base for storing and retrieving different types of project documents.
*   **`config/`**: Contains example configuration files (e.g., `config2.example.yaml`). The active configuration is typically `~/.metagpt/config2.yaml`.
*   **`workspace/`**: The default output directory where MetaGPT saves the projects it generates.
*   **`examples/`**: Provides scripts and examples demonstrating how to use MetaGPT for various tasks or how to extend its functionality (e.g., specific agent setups, game simulations).
*   **`docs/`**: Official documentation for users and contributors.
*   **`tests/`**: Unit and integration tests for the framework.

## 5. Getting Started for Developers

1.  **Prerequisites:**
    *   Python 3.9+ (but less than 3.12)
    *   Node.js and pnpm (for certain features or development tasks, check `README.md`)

2.  **Installation:**
    *   Clone the repository: `git clone https://github.com/geekan/MetaGPT.git`
    *   Navigate to the directory: `cd MetaGPT`
    *   Install in editable mode: `pip install -e .`
    *   Install development dependencies if needed: `pip install -r requirements-dev.txt` (or similar, check project specifics).

3.  **Configuration:**
    *   MetaGPT looks for a configuration file at `~/.metagpt/config2.yaml`.
    *   You can initialize a default one by running `metagpt --init-config`.
    *   Edit this file to add your LLM API keys (e.g., `OPENAI_API_KEY`) and choose your desired model. Refer to `config/config2.example.yaml` for all options.

4.  **Running an Example:**
    *   From the CLI: `metagpt "Create a simple calculator application"`
    *   Check the `workspace/` directory for the generated project.

5.  **Exploring the Code:**
    *   Start by understanding `metagpt/software_company.py` to see the high-level orchestration.
    *   Examine `metagpt/team.py` and the base `metagpt/roles/role.py`.
    *   Look into specific roles like `metagpt/roles/product_manager.py` and their associated actions in `metagpt/actions/`.
    *   Trace the flow of messages and how different roles and actions are triggered.

## 6. Extending MetaGPT

MetaGPT is designed to be extensible. Here are common ways to extend it:

*   **Adding a New Role:**
    1.  Create a new Python file in `metagpt/roles/`.
    2.  Define a class that inherits from `metagpt.roles.role.Role` (or `RoleZero` for more advanced features).
    3.  Implement the `__init__`, `_observe`, `_think`, and `_act` methods.
    4.  Define the role's `name`, `profile`, `goal`, and `constraints`.
    5.  Specify the actions the role should watch using `self._watch({...})`.
    6.  Set the initial actions using `self.set_actions([...])`.
    7.  Integrate the new role into a `Team` in your custom script or by modifying `SoftwareCompany`.

*   **Adding a New Action:**
    1.  Create a new Python file in `metagpt/actions/`.
    2.  Define a class that inherits from `metagpt.actions.action.Action`.
    3.  Implement the `run()` method. This is where the core logic of the action, including any LLM calls, resides.
    4.  Use `ActionNode` if you need structured LLM interaction with prompt templates.
    5.  The action should typically produce an `ActionOutput` or an `AIMessage`.
    6.  Make sure the action can be initialized and called by a `Role`.

*   **Adding a New Tool:**
    1.  Develop your tool as a Python class or set of functions.
    2.  In the `Role` that will use the tool, instantiate it or make its functions available.
    3.  Expose the tool's functionality to the LLM via the role's prompts, often by describing how the LLM can request the tool's use.
    4.  The `RoleZero` class and its `tools` attribute provide a more structured way to integrate and declare tools.

## 7. Debugging and Logging

*   MetaGPT uses Python's `logging` module. You can configure log levels and output in `config2.yaml`.
*   The `logs` directory (usually within `workspace/your_project_name/`) contains detailed logs of agent interactions, LLM inputs/outputs, and actions performed during a run. This is invaluable for debugging.
*   Pay attention to the `max_auto_summarize_code` option in `config2.yaml` or CLI, which can be useful for debugging code generation loops.

By understanding these components and workflows, developers can effectively contribute to, extend, and utilize the MetaGPT framework.
