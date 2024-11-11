# Utilizing Bash to Initialize Users
In this tutorial I will go over how to use simple bash scripts to initalize new users.

User initialization will include:
- Cloning a GitHub repository containing configuration files
- Installing a list of packages defined by the user
- Creating symbolic links to and from our configuration folder
- Adding a new user to the system
- Adding the user to a list of groups
- Giving the user a password

# Cloning our GitHub Repository
Start by installing the git package if you don't already have it:
`sudo pacman -S git`

Then we can clone the repository to our home directory:
`git clone git@gitlab.com:cit2420/2420-as2-starting-files.git`

That's all we need for our config files, now we can move on and create a list of packages we want to install.

# Creating a Package List
Signify which packages you want your new user to install by making a file and typing each package on a new line.

An example file can be seen in this Git repository, it is formatted as follows:
<code>nvim
kakoune
tmux</code>