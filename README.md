# Trying-to-make-a-debug-flag.-It-ain-t-easy...
I've being practicing a lot with case statements since they're a very effective way when passing arguments/flags to a script or function, but I had some what of a problem when dealing with a persistent flag that I like to put in all my scripts, which is a `--debug` | `-D` flag, that basically execute `set -x` to the script, making bash go hiper-verbose, sure I could make a simple `case statement` + `for loop` to parse each flag, but the catch is that each flag is activated by the order written when executing the script.

- Example: Executing `./script.sh --debug --option1 --option2` will give me basically almost every step of the script, including the operation of the other flags, BUT if script if executed with any different order like `./script --option1 --debug --option2` the debug information of the the flag `--option1` will not appear!

The solution I've found was creating a separate parsing of flags that only check for the debug flag right at the beginning of the script:

```sh
    #!/usr/bin/env bash

    for flag in "$@" ; do
    	case "$flag" in
    		-D|--debug )
    			set -x
    			;;
    	esac
    done

    for flag in "$@" ; do
    	case "$flag" in
    		-o1|--option1 )
                # ...
        		;;
        	-o2|--option2 )
                # ...
    	    	;;
            # ...
    	esac
    done
```

But another problem rises, what if an option need a value like for example `--integer N` or `--file /path/file`, I'll be damned! Now `shift` enters the scene:

```sh
            # ...
            # Inside the second parser every flag has shift at the end or in some point of the process
    		-i | --integer )
    			shift
    			int=$1
    			printf "Integer:\t%s\n" "$int"
    			;;
            # ...
```

Again another issue, because of debug special parsing, shift can get loose and not turn the value for the flag the next "`$1`", the solution I've found for an integer flag was this:

```sh
    INTregex='^[0-9]+$' # Outside the parser
            # ...
        	-i | --integer )
        		shift
        		int=$1
        		if [[ ! "$int" =~ $INTregex ]]; then
        			shift
        			int=$1
        			if [[ ! "$int" =~ $INTregex ]]; then
        				printf "ERROR! \nFlag '-i'|'--integer' is not supposed to read chars or floats.\n"
        				exit 1
        			fi
        		fi
            # ...
```

With this basically the problem is solved, but then again, this only corrects integer flags, string or file flgs will need another approach.

## Am I overcomplicating it?

Here the training script I've been working with:

```sh
    #!/usr/bin/env bash

    # 1: Messages meant to read in debug mode (with flag -D|--debug)

    for flag in "$@" ; do
    	case "$flag" in
    		-D|--debug )
    			set -x
    			printf "DEBUG FLAG DETECTED!" &>/dev/null # 1
    			;;
    	esac
    done

    INTregex='^[0-9]+$'
    no_prompt=false
    for flag in "$@" ; do
    	case "$flag" in
    		-i | --integer )
    			printf "INTEGER FLAG DETECTED!" &>/dev/null # 1
    			shift
    			int=$1
    			if [[ ! "$int" =~ $INTregex ]]; then
    				shift
    				int=$1
    				if [[ ! "$int" =~ $INTregex ]]; then
    					printf "ERROR! \nFlag '-i'|'--integer' is not supposed to read chars or floats.\n"
    					exit 1
    				fi
    			fi
    			printf "Integer:\t%s\n" "$int"
    			;;
    		-y | --yes )
    			printf "YES FLAG DETECTED!" &>/dev/null # 1
    			printf "PROMPT WILL NOT BE SHOWN, PLUS SCRIPT WILL GIVE POSITIVE REPLY" &>/dev/null # 1
    			shift
    			no_prompt=true
    			answer=yes
    			;;
    		-n | --no )
    			printf "NO FLAG DETECTED!" &>/dev/null # 1
    			printf "PROMPT WILL NOT BE SHOWN, PLUS SCRIPT WILL GIVE NEGATIVE REPLY" &>/dev/null # 1
    			shift
    			no_prompt=true
    			answer=no
    			;;
    		* )
    			continue
    			;;
    	esac
    done

    while true ; do
    	if [ "$no_prompt" = false ]; then
    		printf "\rWhat is your answer? [Y/n] "; read -rp "" answer
    	else
    		printf "PROMPT SKIPPED!" &>/dev/null # 1
    	fi
    	case "$answer" in
    		[Yy]|Yes|yes )
    			printf "POSITIVE REPLY!" &>/dev/null # 1
    			echo 'Great news!!! Many thanks.'
    			break
    			;;
    		[Nn]|No|no )
    			printf "NEGATIVE REPLY!" &>/dev/null # 1
    			echo 'How dare you?! Defying me like that.'
    			break
    			;;
    		* )
    			printf "WRONG ANSWER!" &>/dev/null # 1
    			echo 'Answer either Yes or No!'
    			sleep 0.6s
    			;;
    	esac
    done
```

