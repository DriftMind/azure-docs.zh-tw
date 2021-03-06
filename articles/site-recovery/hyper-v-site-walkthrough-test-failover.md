---
title: "執行 Hyper-V 複寫至 Azure 的測試容錯移轉 (不含 System Center VMM) | Microsoft Docs"
description: "摘要說明使用 Azure Site Recovery 服務執行 Hyper-V VM 複寫至 Azure 的測試容錯移轉所需的步驟。"
services: site-recovery
documentationcenter: 
author: rayne-wiselman
manager: carmonm
editor: 
ms.assetid: c9a4c3ca-84a0-4668-b700-9b0046202299
ms.service: site-recovery
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: storage-backup-recovery
ms.date: 06/22/2017
ms.author: raynew
ms.translationtype: HT
ms.sourcegitcommit: 83f19cfdff37ce4bb03eae4d8d69ba3cbcdc42f3
ms.openlocfilehash: 0974b9eda2cb7e3ba54a4a0fad0a768db644caf9
ms.contentlocale: zh-tw
ms.lasthandoff: 08/21/2017

---

# <a name="step-11-run-a-test-failover-for-hyper-v-replication-to-azure"></a>步驟 11：執行 Hyper-V 複寫至 Azure 的測試容錯移轉

本文說明如何在 Azure 入口網站中使用 [Azure Site Recovery](site-recovery-overview.md) 服務，從內部部署 Hyper-V 虛擬機器 (非 System Center VMM 所管理) 執行測試容錯移轉。

請在本文下方或 [Azure 復原服務論壇](https://social.msdn.microsoft.com/forums/azure/home?forum=hypervrecovmgr)上張貼意見或問題。

## <a name="before-you-start"></a>開始之前

在您執行測試容錯移轉之前，建議您確認 VM 屬性，並進行任何需要的變更。 您可以在 [複寫的項目] 中存取 VM 屬性。 [程式集]  刀鋒視窗會顯示機器設定與狀態的相關資訊。

## <a name="managed-disk-considerations"></a>受控磁碟的考量

[受控磁碟](../virtual-machines/windows/managed-disks-overview.md)會管理與 VM 磁碟相關聯的儲存體帳戶，從而簡化 Azure VM 的磁碟管理。 

- 只有當容錯移轉至 Azure 時，受控磁碟會加以建立並連結至 VM。 當您啟用保護時，來自內部部署 VM 的資料會複寫到儲存體帳戶中。
- 只有使用 Resource Manager 部署模型部署的 VM 才能建立受控磁碟。
- 具有受控磁碟的電腦目前不支援從 Azure 容錯回復到內部部署 Hyper-V 環境。 若您只要進行移轉 (容錯移轉至 Azure 不含容錯回復)，應該只將 [使用受控磁碟] 設定為 [是]
- 啟用此設定時，只有將 [使用受控磁碟] 啟用的資源群組之可用性設定組可供選取。 具有受控磁碟的 VM 必須位於 [使用受控磁碟] 設定為 [是] 的可用性群組中。 若 VM 未啟用此設定，則只有未啟用受控磁碟的資源群組之可用性設定組可供選取。 [深入了解](https://docs.microsoft.com/azure/virtual-machines/windows/manage-availability#use-managed-disks-for-vms-in-an-availability-set)。
- - 如果您用來複寫的儲存體帳戶已使用「儲存體服務加密」進行加密，就不能在容錯移轉期間建立受控磁碟。 在此情況下，請不要啟用使用受控磁碟的功能，或是停用 VM 保護，然後再將它重新啟用，以使用未啟用加密的儲存體帳戶。 [深入了解](https://docs.microsoft.com/azure/storage/storage-managed-disks-overview#managed-disks-and-encryption)。

 
## <a name="network-considerations"></a>網路考量事項
    
- 您可以將目標 IP 位址設定為要在容錯移轉之後用於 Azure VM。 如果您未提供地址，則容錯移轉的機器會使用 DHCP。 如果您設定無法用於容錯移轉的位址，則容錯移轉會失敗。 如果位址可用於測試容錯移轉網路，則相同的目標 IP 位址可用於測試容錯移轉。
- 網路介面卡的數目會視您指定給目標虛擬機器的大小而有所不同，如下所示：
    - 如果來源電腦上的網路介面卡數目小於或等於針對目標機器大小所允許的介面卡數目，則目標將具備與來源相同的介面卡數目。
    - 如果來源虛擬機器的介面卡數目超過針對目標大小所允許的數目，則將使用目標大小的最大值。
    - 例如，如果來源機器具有兩張網路介面卡，而目標機器大小支援四張，則目標機器將會有兩張介面卡。 如果來源機器具有兩張介面卡，但支援的目標大小僅支援一張，則目標機器將只會有一張介面卡。     
- 如果 VM 有多張網路介面卡，則全部會連接至相同的網路。
- 如果虛擬機器具有多個網路介面卡，則清單中顯示的第一個會變成 Azure 虛擬機器中的*預設*網路介面卡。


## <a name="view-and-manage-vm-settings"></a>檢視及管理 VM 設定

建議您在執行容錯移轉之前，確認來源機器的屬性。

1. 在 [受保護的項目] 中，按一下 [複寫的項目]，然後按一下 [VM]。

    ![啟用複寫](./media/hyper-v-site-walkthrough-test-failover/test-failover1.png)
2. 在 [複寫的項目] 窗格中，您可以看到 VM 資訊、健康情況狀態，以及最新可用復原點的摘要。 如需檢視詳細資訊，請按一下 [屬性]。

    ![啟用複寫](./media/hyper-v-site-walkthrough-test-failover/test-failover2.png)
3. 在 [計算與網路] 中，您可以：
    - 修改 Azure VM 名稱。 名稱必須符合 [Azure 需求](site-recovery-support-matrix-to-azure.md#failed-over-azure-vm-requirements)。
    - 指定容錯移轉後的 [資源群組]。
    - 指定 Azure VM 的目標大小
    - 選取[可用性設定組](../virtual-machines/windows/tutorial-availability-sets.md)。
    - 指定是否要使用[受控磁碟](#managed-disk-considerations)。 如果您想要將受控磁碟連結到您移轉至 Azure 的機器上，請選取 [是]。
    - 檢視或修改網路設定，包括在容錯移轉後 Azure VM 所在的網路/子網路，以及要指派給它的 IP 位址。

    ![啟用複寫](./media/hyper-v-site-walkthrough-test-failover/test-failover4.png)
4. 在 [磁碟] 中，您可以看見 VM 上作業系統和資料磁碟的相關資訊。


## <a name="run-a-test-failover"></a>執行測試容錯移轉

現在，執行測試容錯移轉，確定一切都沒問題。

- 如果您想要在容錯移轉後使用 RDP 連線到 Azure VM，請[準備連線](site-recovery-test-failover-to-azure.md#prepare-to-connect-to-azure-vms-after-failover)。
 - 若要完整測試，您需要在測試環境中將 Active Directory 和 DNS 進行複製。 [深入了解](site-recovery-active-directory.md#test-failover-considerations)。
 - 如需關於測試容錯移轉的完整資訊，請參閱[本文](site-recovery-test-failover-to-azure.md)。
 
 立即執行容錯移轉：

1. 若要容錯移轉單一機器，請在 [複寫的項目] 中，按一下 VM > [+測試容錯移轉] 圖示。
2. 若要容錯移轉復原方案，請在 [復原方案] 中，以滑鼠右鍵按一下方案 > [測試容錯移轉]。 若要建立復原方案，請[遵循這些指示](site-recovery-create-recovery-plans.md)。
3. 在 [測試容錯移轉] 中，選取 Azure VM 在容錯移轉之後要連接的 Azure 網路。
4. 按一下 [確定]  即可開始容錯移轉。 您可以按一下 VM 以開啟其屬性，或在保存庫名稱 > [作業]  > [Site Recovery 作業] 中的 [測試容錯移轉] 作業上按一下，來追蹤進度。
5. 容錯移轉完成之後，您應該也會看到複本 Azure 機器出現在 Azure 入口網站 > [虛擬機器]中。 您應該確定 VM 為適當的大小、已連接到適當的網路，而且正在執行中。
6. 如果您已準備好在容錯移轉後進行連線，應該能夠連線到 Azure VM。
7. 完成後，在復原方案上按一下 [清除測試容錯移轉]。 在 [記事]  中，記錄並儲存關於測試容錯移轉的任何觀察。 這將刪除在測試容錯移轉期間所建立的虛擬機器。



## <a name="next-steps"></a>後續步驟

- [深入了解](site-recovery-failover.md)不同類型的容錯移轉及如何執行。
- [深入了解容錯回復](site-recovery-failback-from-azure-to-hyper-v.md)，以便將 Azure VM 容錯回復和複寫回到主要內部部署 Hyper-V 站台。


