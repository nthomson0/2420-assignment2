# Utilizing Bash to Initialize Users
In this tutorial I will go over how to use simple bash scripts to initalize new users.

User initialization will include:
- Cloning a GitHub repository containing configuration files
- Installing a list of packages defined by the user
- Creating symbolic links to and from our configuration folder
- Adding a new user to the system
- Adding the user to a list of groups
- Giving the user a password

# Project 1: Installing packages and Creating Symbolic Links
In this first step we will work up to the script to install packages for us and create useful symbolic links to a config folder.

## Cloning our GitHub Repositories
Start by installing the git package if you don't already have it:

`sudo pacman -S git`

Then we can clone the configuration repository to our home directory.

`git clone https://gitlab.com/cit2420/2420-as2-starting-files.git`

That's all we need for our config files, now we can clone the scripts from this repository.

`git clone https://github.com/nthomson0/2420-assignment2.git`

## Creating a Package List
Signify which packages you want your new user to install by making a file and typing each package on a new line.

An example file can be seen in this Git repository, it is formatted as follows:

package-list
```
nvim
kakoune
tmux
```

Once you've added the packages you'd like to install we can move to the next step.

## Internal Script 1: package-installer
Within the scripts that we cloned is the package-installer.

Note: Do not run this script directly. It should only be run within `project1`.

This is an extremely simple script that will run interally from our project1 script. It works as follows:
```
#!/bin/bash

# Takes the first argument, checks if it is a valid file.
# If it is a valid file then it will go through the file line by line and install each one with the pacman command.
if [[ -f $1 ]]; then
	for i in $(cat ${1}); do
		pacman -S $i
	done
else
	echo "Please use a valid filepath to a package list."
	exit
fi
```

## Internal Script 2: create-sym-link
The second internal script is create-sym-link which does exactly as it's named.

Note: Do not run this script directly. It should only be run within `project1`.

It creates symbolic links after being given a home directory and configuration directory:
```
#!/bin/bash

# Sets the first argument as the home directory path, second argument as the config directory path.
home_dir=${1}
config_dir=${2}

# Checks if the configuration directory exists.
if [[ ! -d $config_dir ]]; then
	echo "Please enter a valid config directory"
	exit
fi

# Checks if the home directory exists.
if [[ ! -d $home_dir  ]]; then
	echo "Please enter a valid home directory"
	exit
fi

# Creates a symbolic link from ~/bin to ~/<config>/bin only if the link doesnt exist already.
if [[ ! -e ${home_dir}/bin ]]; then
	ln -s ${config_dir}/bin ${home_dir}/bin
else
	echo "The file ${home_dir}/bin already exists, please delete first."
fi

# Creates a symbolic link from ~/.config to ~/<config>/config only if the link doesnt exist already.
if [[ ! -e ${home_dir}/.config ]]; then
	ln -s ${config_dir}/config ${home_dir}/.config
else
	echo "The file ${home_dir}/.config already exists, please delete first."
fi

# Creates a symbolic link from ~/<config>/home/bashrc to ~/.bashrc.
# The given repository contains a blank file called bashrc within /home/ so we just remove that first.
if [[ -f ${repo_dir}/home/bashrc ]]; then
	rm ${repo_dir}/home/bashrc
fi

ln -s ${home_dir}/.bashrc ${repo_dir}/home/bashrc
```

## Running the project1 script
Now that you understand what the internal scripts do, we can run the project 1 script.

The syntax is explained within the script but I'll explain it once more here.

`sudo ./project1 -i 'package-list' -c 'home_dir config_dir'`

Where:
- `sudo` is required, the script will exit if it is not supplied.
- `./project1` is the script filename itself, you can rename this if you desire.
- `-i` installs all packages in the supplied `package-list` argument. The argument is required.
- `-c` creates symbolic links to and from the supplied `config_dir` and `home_dir`. The argument is required.
- `home_dir config_dir` should be supplied as absolute paths, such as /home/arch. Since the file is run with sudo, if you supply a tilde expansion `~` it will automatically expand to the root users home directory which is not what we want.


Note: You will not encounter any errors if all your scripts and text files are in the same location, I recommend simply keeping them in the repository after it's cloned.

```
#!/bin/bash

# Creates variables that we will use in our getopts later.
config_dir=""
home_dir=""
package_list=""

# Requires user to run script as root user.
if [[ $EUID -ne 0 ]]; then
	echo "Please run this script with sudo."
	exit 1
fi

# Prints syntax usage to the user
if [[ -z $1 ]]; then
	echo "
	Syntax: 

	sudo ./project1 -i 'package-list' -c 'home_dir config_dir'

	Options:

	[-i] : Runs package installer for all packages in file provided
	[-c] : Creates symbolic links to config repository folder
	
	Note: When typing in your home_dir and config_dir use an absolute path.
	"
	exit 1
fi

# Checks for options -i or -c which both take arguments.
while getopts ":i:c:" opt; do
	case "${opt}" in
		i)
            # -i will run our package-installer internal script with the supplied package-list as an argument.
			package_list=${OPTARG}
			./package-installer $package_list
			;;
		c)
            # -c will run our create-sym-link internal script with the two supplied paths as arguments.
			path_arr=(${OPTARG})
			home_dir=${path_arr[0]}
			config_dir=${path_arr[1]}
			
			./create-sym-link $home_dir $config_dir
			;;
		:)
            # If no argument is given when using -i or -c it will return an error.
			echo "Error: -{OPTARG} requires an argument"
			exit 1
			;;
		?)
            # If any option other than -i or -c is used the program will exit after running -i or -c.
			exit 1
			;;
	esac
done
```

# Project 2: Initializing a New User
In this step we will go over the project2 script which creates a new user, adds them to groups, and gives them a password.

There are no internal scripts required for this script, we can go right into explaining how it works.

We can start by explaining the syntax of the script:

`sudo ./project2 -u 'user_name' -g 'group1 group2 .. group5' -p`

Where:
- `sudo` is once again required for the script to run. It will exit if not supplied.
- `./project2` is the script filename itself, you can rename this if you desire.
- `-u` will create a user with the specified `user_name`. `-u` and the argument are required for the script to run.
- `-g` is optional and will add the user to the groups in the string supplied.
- `-p` is optional and does not require an argument. It will prompt you for a password to give the new user.

Now we can go into depth of the script itself:
```
#!/bin/bash

# If the script is not run with sudo it will exit.
if [[ $EUID -ne 0 ]]; then
	echo "Please use sudo to run this script."
	exit 1
fi

# If no arguments are supplied then the syntax is given to the user.
if [[ -z $1 ]]; then
	echo "
	Creates a user with the specified username and creates a home directory with a copy of /etc/skel contained.
	Optionally adds them to groups and adds a password.

	Syntax:

	sudo ./project2 -u 'user_name' -g 'group1 group2 .. group5' -p

	Options:

	[-u] : Specified the username to be created
	[-g] : Adds the specified user to the listed groups.
	[-p] : Prompts a password to be attached to the specified user.
	"
	exit 1
fi

# Initializes variables for our getopts.
user_name=""
groups=""
password=0

while getopts ":u:g:p" opt; do
	case "${opt}" in
		u)
            # Sets user_name to the supplied argument after -u
			user_name=${OPTARG}
			;;
		g)
            # Sets groups to the supplied list of arguments after -g
			groups=(${OPTARG})
			;;
		p)
            # Sets password to 1 if the user wishes to add a password by using the -p option
			password=1
			;;
		:)
            # If no arguments are supplied for -u or -g, the program will exit.
			echo "Error: -${OPTARG} requires an argument"
			exit 1
			;;
		?)
            # If any arguments other than -u, -g, or -p are supplied the program will exit.
			exit 1
			;;
	esac
done

# If the user_name field was left empty the script exits.
if [[ -z $user_name ]]; then
	echo "Please enter a username"
	exit 1
fi

# Sets username to first argument, checks if username exists already
if [[ ! -z $(grep $user_name /etc/passwd) ]]; then
	echo "Please pick a unique username."
	exit 1
fi

# Sets the user_id to 1000
declare -i user_id=1000

# If user_id 1000 is taken then it will look for the next available user_id
while : ; do
	if [[ -z $(cat /etc/passwd | grep $user_id) ]]; then
		break
	fi
	user_id=$((user_id+1))
done

# Writes the new user into the /etc/passwd file
cat >> /etc/passwd <<- EOF
	${user_name}:x:${user_id}:${user_id}::/home/${user_name}:/usr/bin/bash
EOF

# Creates the users home directory, copies /etc/skel into that directory
mkdir /home/${user_name}
cp -a /etc/skel/. /home/${user_name}

# By default the user is added to their own group
cat >> /etc/group <<- EOF
	${user_name}:x:${user_id}:
EOF

# The user is also added to the users group by default
user_add=$(grep users: /etc/group)
sed -i "s/${user_add}/${user_add},${user_name}/g" /etc/group

# Iterates through our list of groups we want the user to be added to. If the group exists the user will be added, otherwise it skips the group.
if [[ ! -z $groups ]]; then
	for group in ${groups[@]}; do
		group_add=$(grep "${group}:" /etc/group)

		if [[ ! -z $group_add ]]; then
			sed -i "s/${group_add}/${group_add},${user_name}/g" /etc/group
		fi
	done
fi

# Lastly if the password option was selected then it will prompt the user to add a password.
if [[ $password -eq 1 ]]; then
	passwd ${user_name}
fi
```