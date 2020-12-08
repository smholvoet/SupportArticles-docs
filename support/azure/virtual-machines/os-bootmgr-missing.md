---
title: Troubleshoot Windows VM OS with missing boot manager
description: Explains when the Windows VM boot manager is missing, and how to solve the problem.
ms.date: 12/07/2020
ms.prod-support-area-path: 
ms.reviewer: 
---

# Troubleshoot Windows VM OS with missing boot manager

This article explains when the Windows VM boot manager is missing, and how to solve the problem.

## Symptom

When you pull the screenshot of the VM, the OS boot process is stopped with the error:

```ps
BOOTMGR is missing
Press Ctrl+Alt+Del to restart
```

![Boot Manager is missing](./media/os-bootmgr-missing/1-bootmgr-missing.png)

## Cause

There are several reasons this error could occur:

- The operating system (OS) boot process could not locate an active system partition.
- There's a missing reference on the BCD store to locate the windows partition.
- There has been a Ransomware attack.

## Solution

### Process overview

1. Create and access a Repair VM.
1. Verify that the OS partition is active.
1. Fix the missing reference in the BCD store
1. Verify if a ransomware attack has occurred.
1. Rebuild the VM.

> [!NOTE]
> When encountering this error, the Guest OS is not operational. Troubleshoot this issue in offline mode to resolve this issue.

### Create and access a repair VM

1. Use steps 1-3 of the [VM Repair Commands](https://docs.microsoft.com/azure/virtual-machines/troubleshooting/repair-windows-vm-using-azure-virtual-machine-repair-commands) to prepare a Repair VM.
1. Using Remote Desktop Connection, connect to the Repair VM.

### Verify that the OS partition is active

> [!NOTE]
> This mitigation applies only for Generation 1 VMs. Generation 2 VMs (using UEFI) does not use an active partition

Verify the OS partition which holds the BCD store for the disk is marked as active.

   1. Open an elevated command prompt and open the DISKPART tool.

      `diskpart`

   2. List the disks on the system and look for added disks and proceed to select the new disk. In this example, the new disk is Disk 1.

      ```ps
      list disk
      sel disk 1
      ```

      ![Disk 1](./media/os-bootmgr-missing/11-Gen2-1.png)

   3. List all the partitions on that disk and then proceed to select the partition you want to check. Usually System Managed partitions are smaller and around 350 Mb in size. In the image below, this partition is Partition 1.

      ```ps
      list partition
      sel partition 1
      ```

      ![Partition 1](./media/os-bootmgr-missing/12-Gen2-2.png)

   4. Check the status of the partition. In our example, Partition 1 is not active.

      `detail partition`

      ![Detail Partition](./media/os-bootmgr-missing/13-Gen2-3.png)

      1. If the partition isn't active:

         1. Now change the Active flag and then recheck the change was done properly.

            ```ps
            active
            detail partition
            ```

            ![Active Flag](./media/os-bootmgr-missing/14-Gen2-4.png)

   5. Now exit the DISKPART tool.

      `exit`

### Fix the missing reference in the BCD store

1. Open up an elevated CMD and run CHKDSK on that disks.

   `chkdsk <DRIVE LETTER>: /f`

2. Gather the current boot setup info and document it, take note of the identifier on the active partition.

   1. For Generation 1 VM:

      `bcdedit /store <drive letter>:\boot\bcd /enum`

      1. If this command errors out due to `\boot\bcd` not being found, then go to the following mitigation.

      2. Write down the identifier of the Windows Boot loader. This identifier is the one with the path `\windows\system32\winload.exe`.

         ![Mitigation 2 - Windows Identifier 1](media/os-bootmgr-missing/6-boot configuration data-windows-identifier.png)

   2. For Generation 2 VM:

      `bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /enum`

      1. If this command errors out due to `\boot\bcd` not being found, then go to the following mitigation.

      2. Write down the identifier of the Windows Boot loader. This identifier is the one with the path `\windows\system32\winload.efi`.

         ![Mitigation 2 - Windows Identifier 2](./media/os-bootmgr-missing/15-default-identifier.png)

3. Run the following commands:

   1. For **Generation 1** VM:

      ```ps
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {bootmgr} device partition=<BCD FOLDER - DRIVE LETTER>:
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {bootmgr} integrityservices enable
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} device partition=<WINDOWS FOLDER - DRIVE LETTER>:
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} integrityservices enable
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} recoveryenabled Off
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} osdevice partition=<WINDOWS FOLDER - DRIVE LETTER>:
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} bootstatuspolicy IgnoreAllFailures
      ```

      >[!NOTE]
      In case the VHD has a single partition and both the BCD Folder and Windows Folder are in the same volume, and if the above setup didn't work, then try replacing the partition values with ***boot***.

      ```ps
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {bootmgr} device boot
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {bootmgr} integrityservices enable
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} device boot
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} integrityservices enable
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} recoveryenabled Off
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} osdevice boot
      bcdedit /store <BCD FOLDER - DRIVE LETTER>:\boot\bcd /set {<IDENTIFIER>} bootstatuspolicy IgnoreAllFailures
      ```

   2. For **Generation 2** VM:

      ```ps
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {bootmgr} device partition=<Volume Letter of EFI System Partition>:
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {bootmgr} integrityservices enable
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {<IDENTIFIER>} device partition=<WINDOWS FOLDER - DRIVE LETTER>:
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {<IDENTIFIER>} integrityservices enable
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {<IDENTIFIER>} recoveryenabled Off
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {<IDENTIFIER>} osdevice partition=<WINDOWS FOLDER - DRIVE LETTER>:
      bcdedit /store <Volume Letter of EFI System Partition>:EFI\Microsoft\boot\bcd /set {<IDENTIFIER>} bootstatuspolicy IgnoreAllFailures
      ```

### Verify if a ransomware attack has occurred

If the above solutions did not resolve the issue you may want to verify if a ransomware attack has occurred on your VM.

1. Once the disk is attached, set the Windows Explorer view to show "Hidden items".

2. View the disk and see if there are any .txt files on the root folder with instructions on how to regain access to the files of the VM:

   ![Mitigation 2 - Ransomware](./media/os-bootmgr-missing/17-ransomware.png)

3. If you find a `.txt` file with content that is similar to the below example, it means that the machine was the target of a ransomware attack:

   ```ps
   Your files are now encrypted!
    
   Your personal ID :
    
   "XXXXXXX"
    
   What happened?
    
   Your important documents, databases, documents, network folders are encrypted for your PC security problems.
   No data from your computer has been stolen or deleted.
   Follow the instructions to restore the files.
    
   How to get the automatic decryptor:
    
   1) Contact us by e-mail: xxxxx@xxxx.xxx. In the letter, indicate your personal identifier (look at the beginning of this document)
      and the external ip-address of the computer on which the encrypted files are located.
   2) After answering your request, our operator will give you further instructions that will show what to do next (the answer you will receive as soon as possible)
   ** Second email address xxxxx@xxxx.xxx
    
   Free decryption as guarantee!
   Before paying you can send us up to 3 files for free decryption.
   The total size of files must be less than 10 Mb (non archived), and files should not contain
   valuable information (databases, backups, large excel sheets, etc.).
    
    __________________________________________________________________________________________________
   |                                                                                                  |
   |  How to obtain Bitcoins?                                                                         |
   |                                                                                                  |
   | * The easiest way to buy bitcoins is LocalBitcoins site. You have to register, click             |
   |   'Buy bitcoins', and select the seller by payment method and price:                             |
   |   https://localbitcoins.com/buy_bitcoins                                                         |
   | * Also you can find other places to buy Bitcoins and beginners guide here:                       |
   |   http://www.coindesk.com/information/how-can-i-buy-bitcoins                                     |
   |                                                                                                  |
   |__________________________________________________________________________________________________|
    
    __________________________________________________________________________________________________
   |                                                                                                  |
   | Attention!                                                                                       |
   |                                                                                                  |
   | * Do not rename encrypted files.                                                                 |
   | * Do not try to decrypt your data using third party software, it may cause permanent data loss.  |
   | * Decryption of your files with the help of third parties may cause increased price              |
   |   (they add their fee to our) or you can become a victim of a scam.                              |
   |                                                                                                  |
   |__________________________________________________________________________________________________|
   ```

At this point the VM needs to be restored from backup or rebuilt.  

### Rebuild the VM

Use [step 5 of the VM Repair Commands](https://docs.microsoft.com/azure/virtual-machines/troubleshooting/repair-windows-vm-using-azure-virtual-machine-repair-commands#repair-process-example) to rebuild the VM.

## Next steps

If you still cannot determine the cause of the issue and need more help, you can open a support ticket with Microsoft Customer Support.

If you need more help at any point in this article, you can contact the Azure experts on the [MSDN Azure and Stack Overflow forums](https://azure.microsoft.com/support/forums/). Alternatively, you can file an Azure support incident. Go to the [Azure support site](https://azure.microsoft.com/support/options/), and select **Get support**. For information about using Azure support, read the [Microsoft Azure support FAQ](https://azure.microsoft.com/support/faq/).