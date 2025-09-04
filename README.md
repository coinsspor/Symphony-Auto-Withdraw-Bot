# Symphony Auto-Withdraw Bot

An automated solution for Symphony blockchain validators to withdraw rewards when balance falls below a specified threshold. This bot monitors your wallet balance and automatically executes withdrawal commands when needed.

## Features

- 🔄 Automatic balance monitoring every 30 minutes
- 💰 Auto-withdrawal when balance drops below threshold
- 📝 Detailed logging of all operations
- 🔒 Secure password handling with Expect
- ⚡ Lightweight Python implementation
- 🛡️ Safety checks to prevent unnecessary withdrawals

## Prerequisites

Before installing, ensure you have:

- Ubuntu/Debian-based Linux system
- Symphony node installed and synced
- Python 3.6 or higher
- Root or sudo access
- Active validator node
- Keyring with stored wallet

## Installation

### Step 1: Install Required Dependencies

```bash
# Update system packages
sudo apt update

# Install Python 3 and pip
sudo apt install -y python3 python3-pip

# Install expect for password automation
sudo apt install -y expect

# Install jq for JSON parsing (optional but recommended)
sudo apt install -y jq

# Verify Python version (should be 3.6+)
python3 --version
```

### Step 2: Create the Balance Check Script

Create the main Python script that monitors your balance:

```bash
cat > ~/symphony_check.py << 'EOF'
#!/usr/bin/env python3
import subprocess
import json
import os
from datetime import datetime

# ============ CONFIGURATION - MODIFY THESE VALUES ============
WALLET_ADDR = "symphony1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # Replace with your wallet address
CHAIN_ID = "symphony-1"
MIN_BALANCE = 5000000  # Minimum balance in note (5 MLD = 5,000,000 note)
LOG_FILE = "/root/symphony_withdraw.log"
# ===========================================================

def log_message(message):
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    with open(LOG_FILE, "a") as f:
        f.write(f"[{timestamp}] {message}\n")

def get_balance():
    try:
        cmd = ["symphonyd", "query", "bank", "balances", WALLET_ADDR, "--chain-id", CHAIN_ID, "-o", "json"]
        result = subprocess.run(cmd, capture_output=True, text=True, timeout=10)
        
        # JSON output might be in stderr or stdout
        json_output = result.stderr if result.stderr else result.stdout
        
        if not json_output:
            log_message("Empty output from query")
            return None
            
        data = json.loads(json_output)
        
        for balance in data.get('balances', []):
            if balance.get('denom') == 'note':
                return int(balance.get('amount', 0))
        
        return None
        
    except json.JSONDecodeError as e:
        log_message(f"JSON parse error: {e}")
        return None
    except Exception as e:
        log_message(f"Error: {str(e)}")
        return None

def main():
    log_message("===== CHECK STARTED =====")
    log_message(f"Wallet address: {WALLET_ADDR}")
    log_message("Querying balance...")
    
    balance = get_balance()
    
    if balance is None:
        log_message("ERROR: Could not retrieve balance!")
        log_message("Skipping withdraw for safety")
        log_message("===== CHECK COMPLETED =====\n")
        return
    
    balance_mld = balance / 1000000
    min_balance_mld = MIN_BALANCE / 1000000
    
    log_message(f"Balance retrieved: {balance} note = {balance_mld:.1f} MLD")
    
    if balance < MIN_BALANCE:
        log_message(f"⚠️ Balance below {min_balance_mld} MLD threshold!")
        log_message("Initiating withdrawal...")
        os.system("/root/symphony_withdraw.sh")
    else:
        log_message(f"✅ Balance sufficient: {balance_mld:.1f} MLD (Min: {min_balance_mld} MLD)")
        log_message("No withdrawal needed")
    
    log_message("===== CHECK COMPLETED =====\n")

if __name__ == "__main__":
    main()
EOF

chmod +x ~/symphony_check.py
```

### Step 3: Create the Withdrawal Script

Create the expect script that handles the actual withdrawal:

```bash
cat > ~/symphony_withdraw.sh << 'EOF'
#!/usr/bin/expect -f

# ============ CONFIGURATION - MODIFY THESE VALUES ============
set password "YOUR_KEYRING_PASSWORD_HERE"                           # CRITICAL: Replace with your keyring password
set validator "symphonyvaloperxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"       # Replace with your validator address
set wallet "wallet-name"                                           # Replace with your wallet name
set chain "symphony-1"
# ===========================================================

set timeout 30

proc log_msg {msg} {
    set timestamp [clock format [clock seconds] -format "%Y-%m-%d %H:%M:%S"]
    set fp [open "/root/symphony_withdraw.log" a]
    puts $fp "\[$timestamp\] $msg"
    close $fp
}

log_msg "Starting withdrawal process..."

# Withdraw validator rewards with commission
log_msg "\[1/2\] Withdrawing validator rewards..."
spawn symphonyd tx distribution withdraw-rewards $validator --from $wallet --commission --chain-id $chain --gas auto --gas-adjustment 1.5 --gas-prices 0.025note --yes
expect "Enter keyring passphrase"
send "$password\r"
expect eof

# Wait 5 seconds between transactions
sleep 5

# Withdraw all rewards
log_msg "\[2/2\] Withdrawing all rewards..."
spawn symphonyd tx distribution withdraw-all-rewards --from $wallet --chain-id $chain --gas auto --gas-adjustment 1.5 --gas-prices 0.025note --yes
expect "Enter keyring passphrase"
send "$password\r"
expect eof

log_msg "Withdrawal process completed"
EOF

chmod +x ~/symphony_withdraw.sh
```

### Step 4: Configure Your Settings

#### 4.1 Get Your Required Information

First, gather your wallet and validator information:

```bash
# Get your wallet name and address
symphonyd keys list

# Get your validator address
symphonyd query staking validators | grep "operator_address"
```

#### 4.2 Update the Python Script

Edit the balance check script:

```bash
nano ~/symphony_check.py
```

Update these values:
- `WALLET_ADDR`: Your Symphony wallet address (starts with `symphony1...`)
- `MIN_BALANCE`: Minimum balance threshold in note units
  - 1 MLD = 1,000,000 note
  - 5 MLD = 5,000,000 note (default)
  - 10 MLD = 10,000,000 note

#### 4.3 Update the Withdrawal Script

⚠️ **SECURITY WARNING**: This script will contain your keyring password in plain text. Ensure proper file permissions!

Edit the withdrawal script:

```bash
nano ~/symphony_withdraw.sh
```

Update these values:
- `password`: Your keyring password (KEEP THIS SECURE!)
- `validator`: Your validator address (starts with `symphonyvaloper...`)
- `wallet`: Your wallet name (from `symphonyd keys list`)

### Step 5: Secure Your Scripts

Set proper permissions to protect your password:

```bash
# Make scripts only readable by root
chmod 700 ~/symphony_withdraw.sh
chmod 700 ~/symphony_check.py

# Secure the log file
touch ~/symphony_withdraw.log
chmod 600 ~/symphony_withdraw.log
```

### Step 6: Test the Scripts

Test the balance check:

```bash
python3 ~/symphony_check.py
```

Check the logs:

```bash
tail -20 ~/symphony_withdraw.log
```

You should see something like:
```
[2025-09-04 19:20:31] ===== CHECK STARTED =====
[2025-09-04 19:20:31] Wallet address: symphony1...
[2025-09-04 19:20:31] Querying balance...
[2025-09-04 19:20:31] Balance retrieved: 141118075 note = 141.1 MLD
[2025-09-04 19:20:31] ✅ Balance sufficient: 141.1 MLD (Min: 5.0 MLD)
[2025-09-04 19:20:31] No withdrawal needed
[2025-09-04 19:20:31] ===== CHECK COMPLETED =====
```

### Step 7: Set Up Automatic Execution with Cron

Add the script to crontab for automatic execution:

```bash
crontab -e
```

Add this line at the bottom:

```
*/30 * * * * /usr/bin/python3 /root/symphony_check.py
```

Save and exit (Ctrl+X, then Y, then Enter if using nano).

Verify the cron job was added:

```bash
crontab -l
```

## Testing Withdrawal Function

To test if withdrawal works (without waiting for balance to drop):

```bash
# Temporarily set high threshold (150 MLD)
sed -i 's/MIN_BALANCE = 5000000/MIN_BALANCE = 150000000/g' ~/symphony_check.py

# Run test
python3 ~/symphony_check.py

# Restore original threshold (5 MLD)
sed -i 's/MIN_BALANCE = 150000000/MIN_BALANCE = 5000000/g' ~/symphony_check.py
```

## Understanding the Note/MLD Conversion

Symphony uses "note" as the base unit. The conversion is:
- 1 MLD = 1,000,000 note
- 100 MLD = 100,000,000 note
- 0.5 MLD = 500,000 note

Examples of MIN_BALANCE settings:
- `MIN_BALANCE = 1000000` → 1 MLD threshold
- `MIN_BALANCE = 5000000` → 5 MLD threshold (default)
- `MIN_BALANCE = 10000000` → 10 MLD threshold

## Configuration Options

### Change Check Frequency

To modify how often the script runs, edit crontab:

```bash
crontab -e
```

Common intervals:
- Every 15 minutes: `*/15 * * * *`
- Every 30 minutes: `*/30 * * * *` (default)
- Every hour: `0 * * * *`
- Every 2 hours: `0 */2 * * *`
- Every 6 hours: `0 */6 * * *`
- Once daily at 3 AM: `0 3 * * *`

## Monitoring

### View Logs

Monitor the bot's activity:

```bash
# View last 50 lines
tail -50 ~/symphony_withdraw.log

# Follow logs in real-time
tail -f ~/symphony_withdraw.log

# Count total checks
grep "CHECK STARTED" ~/symphony_withdraw.log | wc -l

# View only withdrawals
grep "Withdrawing" ~/symphony_withdraw.log

# Check for errors
grep "ERROR" ~/symphony_withdraw.log
```

### Monitor System Status

```bash
# Check if cron is running
systemctl status cron

# View cron logs
grep CRON /var/log/syslog | tail -20

# Check script execution history
grep symphony_check /var/log/syslog
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Balance Not Retrieved

If balance cannot be retrieved, check:

```bash
# Node sync status
symphonyd status | jq .SyncInfo.catching_up

# Should return: false
```

Test balance query manually:

```bash
# Basic query
symphonyd query bank balances YOUR_WALLET_ADDRESS --chain-id symphony-1

# JSON format
symphonyd query bank balances YOUR_WALLET_ADDRESS --chain-id symphony-1 -o json
```

#### 2. Withdrawal Fails

Check validator and wallet:

```bash
# Verify validator exists
symphonyd query staking validator YOUR_VALIDATOR_ADDRESS

# Check wallet name
symphonyd keys list

# Test withdrawal manually
symphonyd tx distribution withdraw-rewards YOUR_VALIDATOR --from YOUR_WALLET --commission --chain-id symphony-1 --dry-run
```

#### 3. Script Not Running Automatically

```bash
# Restart cron service
sudo systemctl restart cron

# Check if cron is enabled
sudo systemctl enable cron

# Verify Python path
which python3

# Test script directly
/usr/bin/python3 /root/symphony_check.py
```

#### 4. Permission Denied Errors

```bash
# Fix script permissions
chmod +x ~/symphony_check.py
chmod +x ~/symphony_withdraw.sh

# Fix log file permissions
touch ~/symphony_withdraw.log
chmod 666 ~/symphony_withdraw.log
```

## Security Best Practices

1. **Password Protection**: 
   - Never share your `symphony_withdraw.sh` file
   - Consider using environment variables for passwords
   - Regularly rotate your keyring password

2. **File Permissions**:
   ```bash
   chmod 700 ~/symphony_withdraw.sh  # Only owner can read/execute
   chmod 600 ~/symphony_withdraw.log  # Only owner can read/write
   ```

3. **Regular Monitoring**:
   - Check logs daily for unusual activity
   - Monitor transaction hashes to verify withdrawals

4. **Backup**:
   - Keep secure backups of your wallet keys
   - Document your validator and wallet addresses

## Uninstallation

To completely remove the auto-withdraw bot:

```bash
# 1. Remove from crontab
crontab -e
# Delete the line containing symphony_check.py

# 2. Delete all files
rm ~/symphony_check.py
rm ~/symphony_withdraw.sh
rm ~/symphony_withdraw.log

# 3. Verify removal
ls -la ~/symphony*
```

## Support & Debugging

For additional debugging, enable verbose logging:

```bash
# Add debug output to Python script
sed -i '/def get_balance/a\    log_message("DEBUG: Starting balance query")' ~/symphony_check.py

# Check system logs
journalctl -u cron -f

# Test with verbose output
python3 -v ~/symphony_check.py
```

## License

MIT License - Free to use, modify, and distribute.

## Disclaimer

⚠️ **IMPORTANT**: This bot automates financial operations on your validator node. 
- Always test thoroughly on testnet first
- Monitor logs regularly
- Keep your passwords secure
- The authors are not responsible for any losses

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## Acknowledgments

Created for the Symphony blockchain validator community.

---

**Remember**: Always ensure your Symphony node is fully synced before using this bot. Test all configurations carefully!
