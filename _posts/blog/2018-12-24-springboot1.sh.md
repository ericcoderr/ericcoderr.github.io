#! /bin/bash -

# Set IFS to its default value. The value in the single quote should be <space><tab><newline>.
IFS=$' \t\n'

# Unset all possible aliases. Escape unalias to prevent an alias being used for unalias.
\unalias -a

# Ensure command is not a user function. Special built-in unset will be found before functions.
unset -f awk cat command echo kill printf sleep true

# Put on a reliable PATH prefix.
syspath=$(command -p getconf PATH 2>/dev/null)
if [[ -z "${syspath}" ]]; then
	syspath="/bin:/usr/bin"
fi
PATH="${syspath}:$PATH"
export PATH

# Use ASCII only
LC_ALL=POSIX
export LC_ALL

# print error message
# Usage: error MESSAGE
error()
{
	echo "$@" 1>&2
}

# the shell script name
Program=$(command basename $0)

# Print help info
# Usage: usage
usage()
{
	cat <<- EOF

	Usgae: ${Program} start|stop|restart|status [SpringbootJarName]

	Start/Stop/Restart/Show status of [App].
	Valid operations:
	 	start
	 	    Start the springboot app.

	 	stop
	 	    Stop the springboot app.

	 	restart
	 	    Restart springboot app.

	 	status
	 	    Show the status of the springboot app.

	If no [SpringbootJarName] is specified, start the first .jar in the same directory as the script.

	EOF
}

# Display usage info adn exit with given code
# Usage: usage_and_exit [EXIT_CODE]
usage_and_exit()
{
    usage
    exit $1
}

# check if JAVA_HOME is set.
if [[ -z ${JAVA_HOME} ]]; then
	error "------------------------------------"
	error "JAVA_HOME not set."
	exit 1
fi

# check operation
if [[ -z "$1" ]]; then
	error "------------------------------------"
	printf "No operation specified.\n" 1>&2
	usage_and_exit 1
fi

# the directory containing the script
app_dir=$(command dirname $0)
# the app jar file name
app_jar=$2
# check app
if [[ -z "${app_jar}" ]]; then
	shopt -s nullglob
	apps=("${app_dir}"/*.jar)
	shopt -u nullglob
	if [[ ${#apps[@]} -eq 0 ]]; then
		error "------------------------------------"
		printf "No app specified and no default app found.\n" 1>&2
		usage_and_exit 1
	else
		app_jar=${apps[0]}
	fi
fi

app_jar=$(command readlink -f "${app_jar}")
echo "App: ${app_jar}."

java_bin=${JAVA_HOME}/bin/java
# java start optuions
opts="-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:MaxHeapSize=1024m -XX:ParallelGCThreads=2 -Dspring.config.location=$(command readlink -f $app_dir)/"
log_file="${app_dir}"/start.log

# start app
start()
{
	pid=$(ps -ef | grep "java .* [-]jar ${app_jar}" | awk '{print $2}')
	if [[ -n ${pid} ]]; then
		error "----------------start--------------------"
		error "${app_jar} is running."
	else
		echo "------------------------------------"
		echo "Starting ${app_jar} ..."
		${java_bin} ${opts} -jar "${app_jar}" >> "${log_file}" &
		sleep 1
		pid=$(ps -ef | grep "java .* [-]jar ${app_jar}" | awk '{print $2}')
	fi
	echo "PID: ${pid}."
}


grace_period=30
# stop app
stop()
{
	pid=$(ps -ef | grep "java .* [-]jar ${app_jar}" | awk '{print $2}')
	if [[ -z ${pid} ]]; then
		error "${app_jar} is not running."
	else
		echo "-------------stop-----------------------"
		echo "Stopping ${app_jar}..."
		echo "PID: ${pid}."
		kill "$pid"
		start=$(date +%s)
		end=$((start+grace_period))
		while true; do
			pid=$(ps -ef | grep "java .* [-]jar ${app_jar}" | awk '{print $2}')
			if [[ -z ${pid} ]]; then
				echo "Process ${pid} stopped."
				break;
			fi
			now=$(date +%s)
			if [[ ${now} -ge ${end} ]]; then
				error "${app_jar} has not stopped after ${grace_period} seconds. Kill forceably."
				kill -9 ${pid}
				break
			fi
			sleep 1
		done
	fi
}

# restart app
restart()
{
	stop
	start
}



# show status of app
status()
{
	pid=$(ps -ef | grep "java .* [-]jar ${app_jar}" | awk '{print $2}')
	echo "------------------------------------"
	if [[ -z ${pid} ]]; then
		echo "${app_jar} is not running."
	else
		echo "${app_jar} is running with pid ${pid}."
	fi
}

case "$1" in
	start)
		start
		;;
	stop)
		stop
		;;
	restart)
		restart
		;;
	status)
		status
		;;
	*)
		error "Unsupported operation $1."
		usage_and_exit 1
		;;
esac

