

#### [PATCH v23 0/9] Add ARMv8 RAS virtualization support in QEMU
##### From: Dongjiu Geng <gengdongjiu@huawei.com>

```c
From patchwork Sun Feb 16 10:23:59 2020
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
X-Patchwork-Submitter: Dongjiu Geng <gengdongjiu@huawei.com>
X-Patchwork-Id: 11384355
Return-Path: <SRS0=ZopM=4E=vger.kernel.org=kvm-owner@kernel.org>
Received: from mail.kernel.org (pdx-korg-mail-1.web.codeaurora.org
 [172.30.200.123])
	by pdx-korg-patchwork-2.web.codeaurora.org (Postfix) with ESMTP id 5DCF2924
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 16 Feb 2020 10:22:13 +0000 (UTC)
Received: from vger.kernel.org (vger.kernel.org [209.132.180.67])
	by mail.kernel.org (Postfix) with ESMTP id 4051520857
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 16 Feb 2020 10:22:13 +0000 (UTC)
Received: (majordomo@vger.kernel.org) by vger.kernel.org via listexpand
        id S1726824AbgBPKWA (ORCPT
        <rfc822;patchwork-kvm@patchwork.kernel.org>);
        Sun, 16 Feb 2020 05:22:00 -0500
Received: from szxga05-in.huawei.com ([45.249.212.191]:10183 "EHLO huawei.com"
        rhost-flags-OK-OK-OK-FAIL) by vger.kernel.org with ESMTP
        id S1726020AbgBPKWA (ORCPT <rfc822;kvm@vger.kernel.org>);
        Sun, 16 Feb 2020 05:22:00 -0500
Received: from DGGEMS411-HUB.china.huawei.com (unknown [172.30.72.59])
        by Forcepoint Email with ESMTP id 1E0D65781A35B856D661;
        Sun, 16 Feb 2020 18:21:52 +0800 (CST)
Received: from huawei.com (10.151.151.243) by DGGEMS411-HUB.china.huawei.com
 (10.3.19.211) with Microsoft SMTP Server id 14.3.439.0; Sun, 16 Feb 2020
 18:21:46 +0800
From: Dongjiu Geng <gengdongjiu@huawei.com>
To: <mst@redhat.com>, <imammedo@redhat.com>,
        <xiaoguangrong.eric@gmail.com>, <shannon.zhaosl@gmail.com>,
        <peter.maydell@linaro.org>, <fam@euphon.net>, <rth@twiddle.net>,
        <ehabkost@redhat.com>, <mtosatti@redhat.com>,
        <qemu-devel@nongnu.org>, <kvm@vger.kernel.org>,
        <qemu-arm@nongnu.org>, <pbonzini@redhat.com>,
        <james.morse@arm.com>, <lersek@redhat.com>,
        <jonathan.cameron@huawei.com>,
        <shameerali.kolothum.thodi@huawei.com>
CC: <gengdongjiu@huawei.com>, <zhengxiang9@huawei.com>
Subject: [PATCH v23 0/9] Add ARMv8 RAS virtualization support in QEMU
Date: Sun, 16 Feb 2020 18:23:59 +0800
Message-ID: <20200216102408.22987-2-gengdongjiu@huawei.com>
X-Mailer: git-send-email 2.18.0.huawei.25
In-Reply-To: <20200216102408.22987-1-gengdongjiu@huawei.com>
References: <20200216102408.22987-1-gengdongjiu@huawei.com>
MIME-Version: 1.0
X-Originating-IP: [10.151.151.243]
X-CFilter-Loop: Reflected
Sender: kvm-owner@vger.kernel.org
Precedence: bulk
List-ID: <kvm.vger.kernel.org>
X-Mailing-List: kvm@vger.kernel.org

In the ARMv8 platform, the CPU error types includes synchronous external abort(SEA)
and SError Interrupt (SEI). If exception happens in guest, host does not know the
detailed information of guest, so it is expected that guest can do the recovery.
For example, if an exception happens in a guest user-space application, host does
not know which application encounters errors, only guest knows it.

For the ARMv8 SEA/SEI, KVM or host kernel delivers SIGBUS to notify userspace.
After user space gets the notification, it will record the CPER into guest GHES
buffer and inject an exception or IRQ to guest.

In the current implementation, if the type of SIGBUS is BUS_MCEERR_AR, we will
treat it as a synchronous exception, and notify guest with ARMv8 SEA
notification type after recording CPER into guest.

A) This series of patches are based on Qemu 4.2, which include two parts:
1. Generate APEI/GHES table.
2. Handle the SIGBUS signal, record the CPER in runtime and fill it into guest
   memory, then notify guest according to the type of SIGBUS.

B) The solution was suggested by James(james.morse@arm.com); The APEI part solution was suggested by Laszlo(lersek@redhat.com). Show some discussions in [1].

C) This series of patches have already been tested on ARM64 platform with RAS
feature enabled:
1. Show the APEI part verification result in [2].
2. Show the SIGBUS of BUS_MCEERR_AR handling verification result in [3].

D) Add 'ras' option in command Line to enable guest RAS error recovery feature, For example:
KVM model: ./qemu-system-aarch64 --enable-kvm -cpu host --bios QEMU_EFI.fd_new  -machine virt,gic-version=3,ras,kernel-irqchip=on  -smp 4 -nographic -kernel Image  -append "rdinit=/init console=ttyAMA0 mem=512M root=/dev/ram0" -initrd guestfs_new.cpio.gz
TCG model: ./qemu-system-aarch64 -cpu cortex-a57 --bios QEMU_EFI.fd_new  -machine virt,gic-version=3,ras,kernel-irqchip=on  -smp 4 -nographic -kernel Image  -append "rdinit=/init console=ttyAMA0 mem=512M root=/dev/ram0" -initrd guestfs_new.cpio.gz
---
Change since v22:
1. Using 1 * KiB instead of 0x400 to define max size of one error block
2. Make the alignment to 8 bytes in bios_linker_loader_alloc()
3. Change "Copyright (c) 2019" to "Copyright (c) 2020" in file header
4. Fix some code style warnings/errors and add some comments in code
5. Address Jonathan's comments to easily support CCIX error injection
6. Add vmstate_ghes_state .subsections in vmstate_acpi_ged

Change since v21:
1. Make the user-facing 'ras' option description more clearly to address Peter's comments.
2. Update the doc description in "docs/specs/acpi_hest_ghes.rst"
3. Split HEST/GHES patches to more patches to make the review easily
4. Using source_id to index the location to save the CPER.
5. Optimize and simplify the logic to build HEST/GHES table to address Igor/Michael/Beata comments.
6. make ghes_addr_le a part of GED device.

Change since v20:
1. Move some implementation details from acpi_ghes.h to acpi_ghes.c
2. Add the reviewers for the ACPI/APEI/GHES part

Change since v19:
1. Fix clang compile error
2. Fix sphinx build error

Change since v18:
1. Fix some code-style and typo/grammar problems.
2. Remove no_ras in the VirtMachineClass struct.
3. Convert documentation to rst format.
4. Simplize the code and add comments for some magic value.
5. Move kvm_inject_arm_sea() function into the patch where it's used.
6. Register the reset handler(kvm_unpoison_all()) in the kvm_init() function.

Change since v17:
1. Improve some commit messages and comments.
2. Fix some code-style problems.
3. Add a *ras* machine option.
4. Move HEST/GHES related structures and macros into "hw/acpi/acpi_ghes.*".
5. Move HWPoison page functions into "include/sysemu/kvm_int.h".
6. Fix some bugs.
7. Improve the design document.

Change since v16:
1. check whether ACPI table is enabled when handling the memory error in the SIGBUS handler.

Change since v15:
1. Add a doc-comment in the proper format for 'include/exec/ram_addr.h'
2. Remove write_part_cpustate_to_list() because there is another bug fix patch
   has been merged "arm: Allow system registers for KVM guests to be changed by QEMU code"
3. Add some comments for kvm_inject_arm_sea() in 'target/arm/kvm64.c'
4. Compare the arm_current_el() return value to 0,1,2,3, not to PSTATE_MODE_* constants.
5. Change the RAS support wasn't introduced before 4.1 QEMU version.
6. Move the no_ras flag  patch to begin in this series

Change since v14:
1. Remove the BUS_MCEERR_AO handling logic because this asynchronous signal was masked by main thread
2. Address some Igor Mammedov's comments(ACPI part)
   1) change the comments for the enum AcpiHestNotifyType definition and remove ditto in patch 1
   2) change some patch commit messages and separate "APEI GHES table generation" patch to more patches.
3. Address some peter's comments(arm64 Synchronous External Abort injection)
   1) change some code notes
   2) using arm_current_el() for current EL
   2) use the helper functions for those (syn_data_abort_*).

Change since v13:
1. Move the patches that set guest ESR and inject virtual SError out of this series
2. Clean and optimize the APEI part patches
3. Update the commit messages and add some comments for the code

Change since v12:
1. Address Paolo's comments to move HWPoisonPage definition to accel/kvm/kvm-all.c
2. Only call kvm_cpu_synchronize_state() when get the BUS_MCEERR_AR signal
3. Only add and enable GPIO-Signal and ARMv8 SEA two hardware error sources
4. Address Michael's comments to not sync SPDX from Linux kernel header file

Change since v11:
Address James's comments(james.morse@arm.com)
1. Check whether KVM has the capability to to set ESR instead of detecting host CPU RAS capability
2. For SIGBUS_MCEERR_AR SIGBUS, use Synchronous-External-Abort(SEA) notification type
   for SIGBUS_MCEERR_AO SIGBUS, use GPIO-Signal notification


Address Shannon's comments(for ACPI part):
1. Unify hest_ghes.c and hest_ghes.h license declaration
2. Remove unnecessary including "qmp-commands.h" in hest_ghes.c
3. Unconditionally add guest APEI table based on James's comments(james.morse@arm.com)
4. Add a option to virt machine for migration compatibility. On new virt machine it's on
   by default while off for old ones, we enabled it since 2.12
5. Refer to the ACPI spec version which introduces Hardware Error Notification first time
6. Add ACPI_HEST_NOTIFY_RESERVED notification type

Address Igor's comments(for ACPI part):
1. Add doc patch first which will describe how it's supposed to work between QEMU/firmware/guest
   OS with expected flows.
2. Move APEI diagrams into doc/spec patch
3. Remove redundant g_malloc in ghes_record_cper()
4. Use build_append_int_noprefix() API to compose whole error status block and whole APEI table,
   and try to get rid of most structures in patch 1, as they will be left unused after that
5. Reuse something like https://github.com/imammedo/qemu/commit/3d2fd6d13a3ea298d2ee814835495ce6241d085c
   to build GAS
6. Remove much offsetof() in the function
7. Build independent tables first and only then build dependent tables passing to it pointers
   to previously build table if necessary.
8. Redefine macro GHES_ACPI_HEST_NOTIFY_RESERVED to ACPI_HEST_ERROR_SOURCE_COUNT to avoid confusion


Address Peter Maydell's comments
1. linux-headers is done as a patch of their own created using scripts/update-linux-headers.sh run against a
   mainline kernel tree
2. Tested whether this patchset builds OK on aarch32
3. Abstract Hwpoison page adding code  out properly into a cpu-independent source file from target/i386/kvm.c,
   such as kvm-all.c
4. Add doc-comment formatted documentation comment for new globally-visible function prototype in a header


---
[1]:
https://lkml.org/lkml/2017/2/27/246
https://patchwork.kernel.org/patch/9633105/
https://patchwork.kernel.org/patch/9925227/

[2]:
Note: the UEFI(QEMU_EFI.fd) is needed if guest want to use ACPI table.

After guest boot up, dump the APEI table, then can see the initialized table
(1) # iasl -p ./HEST -d /sys/firmware/acpi/tables/HEST
(2) # cat HEST.dsl
    /*
     * Intel ACPI Component Architecture
     * AML/ASL+ Disassembler version 20170728 (64-bit version)
     * Copyright (c) 2000 - 2017 Intel Corporation
     *
     * Disassembly of /sys/firmware/acpi/tables/HEST, Mon Sep  5 07:59:17 2016
     *
     * ACPI Data Table [HEST]
     *
     * Format: [HexOffset DecimalOffset ByteLength]  FieldName : FieldValue
     */

    ..................................................................................
    [308h 0776   2]                Subtable Type : 000A [Generic Hardware Error Source V2]
    [30Ah 0778   2]                    Source Id : 0001
    [30Ch 0780   2]            Related Source Id : FFFF
    [30Eh 0782   1]                     Reserved : 00
    [30Fh 0783   1]                      Enabled : 01
    [310h 0784   4]       Records To Preallocate : 00000001
    [314h 0788   4]      Max Sections Per Record : 00000001
    [318h 0792   4]          Max Raw Data Length : 00001000

    [31Ch 0796  12]         Error Status Address : [Generic Address Structure]
    [31Ch 0796   1]                     Space ID : 00 [SystemMemory]
    [31Dh 0797   1]                    Bit Width : 40
    [31Eh 0798   1]                   Bit Offset : 00
    [31Fh 0799   1]         Encoded Access Width : 04 [QWord Access:64]
    [320h 0800   8]                      Address : 00000000785D0040

    [328h 0808  28]                       Notify : [Hardware Error Notification Structure]
    [328h 0808   1]                  Notify Type : 08 [SEA]
    [329h 0809   1]                Notify Length : 1C
    [32Ah 0810   2]   Configuration Write Enable : 0000
    [32Ch 0812   4]                 PollInterval : 00000000
    [330h 0816   4]                       Vector : 00000000
    [334h 0820   4]      Polling Threshold Value : 00000000
    [338h 0824   4]     Polling Threshold Window : 00000000
    [33Ch 0828   4]        Error Threshold Value : 00000000
    [340h 0832   4]       Error Threshold Window : 00000000

    [344h 0836   4]    Error Status Block Length : 00001000
    [348h 0840  12]            Read Ack Register : [Generic Address Structure]
    [348h 0840   1]                     Space ID : 00 [SystemMemory]
    [349h 0841   1]                    Bit Width : 40
    [34Ah 0842   1]                   Bit Offset : 00
    [34Bh 0843   1]         Encoded Access Width : 04 [QWord Access:64]
    [34Ch 0844   8]                      Address : 00000000785D0098

    [354h 0852   8]            Read Ack Preserve : 00000000FFFFFFFE
    [35Ch 0860   8]               Read Ack Write : 0000000000000001

    .....................................................................................

(3) After a synchronous external abort(SEA) happen, Qemu receive a SIGBUS and 
    filled the CPER into guest GHES memory.  For example, according to above table,
    the address that contains the physical address of a block of memory that holds
    the error status data is 0x00000000785D0040
(4) the address of error source which is SEA notification type is 0x785d80b0
    (qemu) xp /1 0x00000000785D0040
    00000000785d0040: 0x785d80b0

(5) check the content of generic error status block and generic error data entry
    (qemu) xp /100x 0x785d80b0
    00000000785d80b0: 0x00000001 0x00000000 0x00000000 0x00000098
    00000000785d80c0: 0x00000000 0xa5bc1114 0x4ede6f64 0x833e63b8
    00000000785d80d0: 0xb1837ced 0x00000000 0x00000300 0x00000050
    00000000785d80e0: 0x00000000 0x00000000 0x00000000 0x00000000
    00000000785d80f0: 0x00000000 0x00000000 0x00000000 0x00000000
    00000000785d8100: 0x00000000 0x00000000 0x00000000 0x00004002
(6) check the OSPM's ACK value(for example SEA)
    /* Before OSPM acknowledges the error, check the ACK value */
    (qemu) xp /1 0x00000000785D0098
    00000000785d00f0: 0x00000000

    /* After OSPM acknowledges the error, check the ACK value, it change to 1 from 0 */
    (qemu) xp /1 0x00000000785D0098
    00000000785d00f0: 0x00000001

[3]: KVM deliver "BUS_MCEERR_AR" to Qemu, Qemu record the guest CPER and inject
    synchronous external abort to notify guest, then guest do the recovery.

[ 1552.516170] Synchronous External Abort: synchronous external abort (0x92000410) at 0x000000003751c6b4
[ 1553.074073] {1}[Hardware Error]: Hardware error from APEI Generic Hardware Error Source: 8
[ 1553.081654] {1}[Hardware Error]: event severity: recoverable
[ 1554.034191] {1}[Hardware Error]:  Error 0, type: recoverable
[ 1554.037934] {1}[Hardware Error]:   section_type: memory error
[ 1554.513261] {1}[Hardware Error]:   physical_address: 0x0000000040fa6000
[ 1554.513944] {1}[Hardware Error]:   error_type: 0, unknown
[ 1555.041451] Memory failure: 0x40fa6: Killing mca-recover:1296 due to hardware memory corruption
[ 1555.373116] Memory failure: 0x40fa6: recovery action for dirty LRU page: Recovered

Dongjiu Geng (9):
  hw/arm/virt: Introduce a RAS machine option
  docs: APEI GHES generation and CPER record description
  ACPI: Build related register address fields via hardware error fw_cfg
    blob
  ACPI: Build Hardware Error Source Table
  ACPI: Record the Generic Error Status Block address
  KVM: Move hwpoison page related functions into kvm-all.c
  ACPI: Record Generic Error Status Block(GESB) table
  target-arm: kvm64: handle SIGBUS signal from kernel or KVM
  MAINTAINERS: Add ACPI/HEST/GHES entries

 MAINTAINERS                            |   9 +
 accel/kvm/kvm-all.c                    |  36 +++
 default-configs/arm-softmmu.mak        |   1 +
 docs/specs/acpi_hest_ghes.rst          | 110 ++++++++
 docs/specs/index.rst                   |   1 +
 hw/acpi/Kconfig                        |   4 +
 hw/acpi/Makefile.objs                  |   1 +
 hw/acpi/aml-build.c                    |   2 +
 hw/acpi/generic_event_device.c         |  18 ++
 hw/acpi/ghes.c                         | 450 +++++++++++++++++++++++++++++++++
 hw/arm/virt-acpi-build.c               |  17 ++
 hw/arm/virt.c                          |  23 ++
 include/hw/acpi/aml-build.h            |   1 +
 include/hw/acpi/generic_event_device.h |   2 +
 include/hw/acpi/ghes.h                 |  74 ++++++
 include/hw/arm/virt.h                  |   1 +
 include/sysemu/kvm.h                   |   3 +-
 include/sysemu/kvm_int.h               |  12 +
 target/arm/cpu.h                       |   4 +
 target/arm/helper.c                    |   2 +-
 target/arm/internals.h                 |   5 +-
 target/arm/kvm64.c                     |  73 ++++++
 target/arm/tlb_helper.c                |   2 +-
 target/i386/cpu.h                      |   2 +
 target/i386/kvm.c                      |  36 ---
 25 files changed, 847 insertions(+), 42 deletions(-)
 create mode 100644 docs/specs/acpi_hest_ghes.rst
 create mode 100644 hw/acpi/ghes.c
 create mode 100644 include/hw/acpi/ghes.h
```
#### [PATCH 0/5] Random debug/PMU fixes for 5.6
##### From: Marc Zyngier <maz@kernel.org>
X-SA-Exim-Mail-From: maz@kernel.org

```c
From patchwork Sun Feb 16 18:53:19 2020
Content-Type: text/plain; charset="utf-8"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
X-Patchwork-Submitter: Marc Zyngier <maz@kernel.org>
X-Patchwork-Id: 11384599
Return-Path: <SRS0=ZopM=4E=vger.kernel.org=kvm-owner@kernel.org>
Received: from mail.kernel.org (pdx-korg-mail-1.web.codeaurora.org
 [172.30.200.123])
	by pdx-korg-patchwork-2.web.codeaurora.org (Postfix) with ESMTP id D9574139A
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 16 Feb 2020 18:53:53 +0000 (UTC)
Received: from vger.kernel.org (vger.kernel.org [209.132.180.67])
	by mail.kernel.org (Postfix) with ESMTP id B9BDB20857
	for <patchwork-kvm@patchwork.kernel.org>;
 Sun, 16 Feb 2020 18:53:53 +0000 (UTC)
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=kernel.org;
	s=default; t=1581879233;
	bh=CDMcIuGZauuEKnBrc2rOLv93aehoxJbaP6h3f0RnX18=;
	h=From:To:Cc:Subject:Date:List-ID:From;
	b=YjmBd5/DpLRRFzRLUA/Rxz9cqZS1gGlW7fHB60ECg6LEwMgGMfYUc0vmCL8eTNoyL
	 K6PHgKTdmqxgw0/rCvPoxwFwSV50HoX+RluBhRU+6ldFMXyzWVRiqSmSIdwh3hXLWY
	 OhZKxxVEKRiEDFtYgBrEN+nuMyLkEM/DF2cC2Ato=
Received: (majordomo@vger.kernel.org) by vger.kernel.org via listexpand
        id S1726256AbgBPSxk (ORCPT
        <rfc822;patchwork-kvm@patchwork.kernel.org>);
        Sun, 16 Feb 2020 13:53:40 -0500
Received: from mail.kernel.org ([198.145.29.99]:38416 "EHLO mail.kernel.org"
        rhost-flags-OK-OK-OK-OK) by vger.kernel.org with ESMTP
        id S1725989AbgBPSxk (ORCPT <rfc822;kvm@vger.kernel.org>);
        Sun, 16 Feb 2020 13:53:40 -0500
Received: from disco-boy.misterjones.org (disco-boy.misterjones.org
 [51.254.78.96])
        (using TLSv1.2 with cipher ECDHE-RSA-AES256-GCM-SHA384 (256/256 bits))
        (No client certificate requested)
        by mail.kernel.org (Postfix) with ESMTPSA id A672E20857;
        Sun, 16 Feb 2020 18:53:39 +0000 (UTC)
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/simple; d=kernel.org;
        s=default; t=1581879219;
        bh=CDMcIuGZauuEKnBrc2rOLv93aehoxJbaP6h3f0RnX18=;
        h=From:To:Cc:Subject:Date:From;
        b=hcLuvQ6t0pWiScsDk331LZ8xUf9I4J6C+AiHvBLI3vBXX3yrK/XbyHYS20LSJm7YS
         X6PJnMmZQk1psVCfFEa5F8a92HT7yAYyrI5or5hNtb/dYcsUpg9QytOly26kPX5D2n
         0ayVE3bL7IL1dSbRJJwUUkf/Vakab8Ine9qDJ0Wc=
Received: from 78.163-31-62.static.virginmediabusiness.co.uk ([62.31.163.78]
 helo=why.lan)
        by disco-boy.misterjones.org with esmtpsa
 (TLS1.3:ECDHE_RSA_AES_256_GCM_SHA384:256)
        (Exim 4.92)
        (envelope-from <maz@kernel.org>)
        id 1j3P2v-005iWD-O3; Sun, 16 Feb 2020 18:53:37 +0000
From: Marc Zyngier <maz@kernel.org>
To: linux-arm-kernel@lists.infradead.org, kvmarm@lists.cs.columbia.edu,
        kvm@vger.kernel.org
Cc: Peter Maydell <peter.maydell@linaro.org>,
        James Morse <james.morse@arm.com>,
        Julien Thierry <julien.thierry.kdev@gmail.com>,
        Suzuki K Poulose <suzuki.poulose@arm.com>
Subject: [PATCH 0/5] Random debug/PMU fixes for 5.6
Date: Sun, 16 Feb 2020 18:53:19 +0000
Message-Id: <20200216185324.32596-1-maz@kernel.org>
X-Mailer: git-send-email 2.20.1
MIME-Version: 1.0
X-SA-Exim-Connect-IP: 62.31.163.78
X-SA-Exim-Rcpt-To: linux-arm-kernel@lists.infradead.org,
 kvmarm@lists.cs.columbia.edu, kvm@vger.kernel.org, peter.maydell@linaro.org,
 james.morse@arm.com, julien.thierry.kdev@gmail.com, suzuki.poulose@arm.com
X-SA-Exim-Mail-From: maz@kernel.org
X-SA-Exim-Scanned: No (on disco-boy.misterjones.org);
 SAEximRunCond expanded to false
Sender: kvm-owner@vger.kernel.org
Precedence: bulk
List-ID: <kvm.vger.kernel.org>
X-Mailing-List: kvm@vger.kernel.org

Here's a few cleanups and fixes for 5.6, all around our debug and
PMU ID register emulation:

1) Missing RES1 bit in DBGDIDR
2) Limiting PMU version to v8.1
3) Limiting debug version to ARMv8.0
4) Support for ARMv8.4-PMU

(1) was reported by Peter, (2) had a patch from Andrew Murray on the
list, but it's been a while since he was going to rebase and fix it,
so I did bite the bullet. (3) is needed until we implement the right
thing with NV. (4) was too easy not to be done right away.

Patch #2 is a cleanup that helps the following patches.

Unless someone objects, I'd like to take this into 5.6 (except maybe
for the last patch).

Marc Zyngier (5):
  KVM: arm64: Fix missing RES1 in emulation of DBGBIDR
  KVM: arm64: Refactor filtering of ID registers
  kvm: arm64: Limit PMU version to ARMv8.1
  KVM: arm64: Limit the debug architecture to ARMv8.0
  KVM: arm64: Upgrade PMU support to ARMv8.4

 arch/arm64/include/asm/sysreg.h |  2 ++
 arch/arm64/kvm/sys_regs.c       | 35 +++++++++++++++++++++++++--------
 2 files changed, 29 insertions(+), 8 deletions(-)
```