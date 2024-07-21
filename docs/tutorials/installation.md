# 🌐  Installation

## Developer Installation

Trace itself is a minimal package that only relies on Python.
The ability to capture execution trace of Python program is defined in `opto.trace` and does not rely on
any external dependencies.

However, if you want to use optimizer `opto.optimizers`, then we require `autogen` package to manage API requests and communications with LLM APIs.

```{admonition} Installation Command
The installation script will git clone a version of AutoGen. 
You may require Git Large File Storage if git is unable to clone the repository otherwise.

```bash
git clone https://github.com/microsoft/Trace.git
cd Trace
pip install -e .
```