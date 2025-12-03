# USB-Linux-Optimiser
This repository is a composition of multiple shell scripts included as steps in building and optimising a LiveUSB install.

```bash
#!/bin/bash

#!/bin/bash

# Ensure the user has dialog installed
if ! command -v dialog &> /dev/null; then
    echo "dialog is required for this script. Installing..."
    sudo apt-get install -y dialog || { echo "Failed to install dialog"; exit 1; }
fi

# Step 1: Prompt the user for the internal drive name
echo "Please enter the internal drive name (e.g., /dev/sda1):"
read -r INTERNAL_DRIVE


# Ensure the drive exists
if [ ! -e "$INTERNAL_DRIVE" ]; then
    echo "Error: $INTERNAL_DRIVE does not exist. Please check the drive name and try again."
    exit 1
fi

# Step 2: Mount the internal drive to /mnt
echo "Mounting internal drive $INTERNAL_DRIVE to /mnt..."


sudo mount "$INTERNAL_DRIVE" /mnt || { echo "Failed to mount $INTERNAL_DRIVE"; exit 1; }


# Step 3: Use dialog to choose which /var directories to move
CHOICES=$(dialog --checklist "Select the directories to move to internal storage:" 15 60 4 \
    1 "/var/cache" off \
    2 "/var/log" off \
    3 "/var/tmp" off \
    4 "/var/lib" off \
    3>&1 1>&2 2>&3)

# Check if any directories were selected
if [ -z "$CHOICES" ]; then
    echo "No directories selected. Exiting."
    exit 1
fi


# Step 4,5: Move the selected directories
for DIR in $CHOICES; do
    TARGET_DIR="/mnt$(basename $DIR)"
    echo "Moving $DIR to $TARGET_DIR..."
    
    sudo mv "$DIR" "$TARGET_DIR" || { echo "Failed to move $DIR"; exit 1; }
    
    # Step 5: Create symbolic link for the moved directory
    echo "Creating symbolic link for $DIR..."
    sudo ln -s "$TARGET_DIR" "$DIR" || { echo "Failed to create symbolic link for $DIR"; exit 1; }
    
    echo "$DIR has been moved to $TARGET_DIR and linked."
done

# Step 6: Update /etc/fstab to ensure these directories are mounted on boot
echo "Updating /etc/fstab to ensure directories are mounted on boot..."

for DIR in $CHOICES; do
    TARGET_DIR="/mnt$(basename $DIR)"
    echo "$INTERNAL_DRIVE   $TARGET_DIR   ext4   defaults   0   2" | sudo tee -a /etc/fstab
done

# Final message
echo "Your var is now on internal media."
echo "If you reboot now, everything will return to undefined state"
