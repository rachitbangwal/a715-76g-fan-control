# ***\*Note*** :-
- All values are in hex, if not explicitly mentioned.
- `ec-probe` command is provided by notebook fan control ***\*nbfc-revive***.
- Tested with nbfc-revive 1.7.1 and confirmed working.


# Control Registers
1. ***Acer power profile: `0x54` (FN+F)***
	- Decompiled DSDT flags this register as `FANT`, refering to fan table.
	- `0x54` EC register is used for storing `FN + f` profiles in memory with values ranging from 0 - 2, mapping to below corresponding register values :-
		- silent:- `0x01`
		- normal:- `0x00`
		- performance:- `0x02`
3. ***Fan speed: `0x55`***
	- `0x55` is the EC register used for controlling fan speed which has 10 steps ranging from `0x00` to `0x0A`.
	- All control commands sems to be provided to left fan first and then right fan seems to be matching this speed ***(Verification needed that 2nd fan register exists or not)***
	- On min `0x00` state causes "stuck loop" where left fan turns off and then restarts and turns off again and repeats this state while right fan seems to be spinning at low rpm.
 	- This register cannot retain values as most probably EC is in Auto Mode. 
4. ***EC thermal register: `0x58`***
	- This register is used as a store for modulating the fan speed according to current HW reported temps.
	- This register is EC specific and is not shared/exposed to rest of the OS ***(Verification needed)***.
	- Can turn of the Ideating fans at `0x01` if set this register value to a lower threshold lower than 45~50 degree Celsius ***\* in decimal***.
	- ***WARNING*** :- Only use this register for testing purpose.


# Debugging ***(PowerShell script)***
1. Monitor Live update of `0x54 0x55 0x58` registers:-
	``` ps1
	while($true) {
    $val54 = (ec-probe read 0x54).Split(' ')[-1]
    $val55 = (ec-probe read 0x55).Split(' ')[-1]
    $temp = (ec-probe read 0x58).Split(' ')[-1]
    
    Clear-Host
    Write-Host "--- Acer Aspire 7 EC Monitor ---" -ForegroundColor Cyan
    Write-Host "Manual Mode (0x54): $val54"
    Write-Host "Fan Speed   (0x55): $val55"
    Write-Host "CPU Temp    (0x58): $temp (Hex)"
    Write-Host "--------------------------------"
    Start-Sleep -Milliseconds 200
    }
	```
1. Update registers forcefully using loop:
	- Repeatedly override and force custom fan speeds manually ***\*below example sets profile to performance (0x02) and forces max speed [0x01 (min), 0x0A(max), 0x00(min but, bugged in silent mode)]***: 
		```ps1
		 do { ec-probe write 0x54 0x02; ec-probe write 0x55 0xA } while ($true)
		```
	- EC temperature to trick EC to fully turn of fans:
		```ps1
		 do { ec-probe write 0x54 0x02; ec-probe write 0x58 0x14 } while ($true)
		```

# Speculations:
- `0x4f`:-
	- ***IN LINUX*** Writing values ranging from 0 to 11 to this register *right fan* correctly responds to out inputs and stays to that values.
	- Changes made to `0x55` register propagates to this register too.
	- Changes on `0x55` register overrides the manual changes made to this register and then it seems to enter auto mode where `0x55` controls this register.

-  RPM:
	- `0x6A` MSB & `0x6B` LSB:
		- These 2 register seems to respond to when manually changing values of `0x4f` register, as if register set to 0 these both changes to 0 and same for other values.
	- `0x5c` MSB & `0x5d` LSB:
		- While '0x55' register cannot retain the manually set value but these 2 register seems to respond to speed change in `0x55` register.

# References
1. [IT5570-128 EC controller](https://ia800805.us.archive.org/18/items/it-5570-a-v-0.3.1-u/IT5570_A_V0.3.1_U.pdf)
2. [Other laptop Service Manuel using same EC](https://www.mikrocontroller.net/attachment/609783/nightsky_arx15.pdf)
3. [Notebook Fan control: nbfc-revive](https://github.com/UraniumDonut/nbfc-revive)
