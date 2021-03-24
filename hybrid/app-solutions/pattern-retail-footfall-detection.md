---
title: Vzor detekce Footfall pomocí Azure a centra Azure Stack
description: Naučte se používat Azure a centrum Azure Stack k implementaci řešení pro detekci Footfall založeného na AI pro analýzu provozu v maloobchodním obchodě.
author: BryanLa
ms.topic: article
ms.date: 10/31/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 10/31/2019
ms.openlocfilehash: 866557ec3af2337e9f034da84cf417675508563b
ms.sourcegitcommit: 962334135b63ac99c715e7bc8fb9282648ba63c9
ms.translationtype: MT
ms.contentlocale: cs-CZ
ms.lasthandoff: 03/23/2021
ms.locfileid: "104895325"
---
# <a name="footfall-detection-pattern"></a>Model detekce Footfall

Tento model poskytuje přehled implementace řešení detekce Footfall založených na AI pro analýzu provozu návštěvníků v prodejnách. Řešení generuje přehledy z reálných akcí, pomocí Azure, centra Azure Stack a Custom Vision AI pro vývojáře.

## <a name="context-and-problem"></a>Kontext a problém

Obchody společnosti Contoso by chtěli získat přehled o tom, jak zákazníci dostávají své aktuální produkty ve vztahu k uložení rozložení. Nemůžou do každého oddílu umístit pracovníky a je neefektivní, aby tým analytiků provedl kontrolu celého záběru na kameře v celém úložišti. Kromě toho žádný z jejich úložišť nemá dostatek šířky pásma pro streamování videa ze všech kamer do cloudu pro účely analýzy.

Společnost Contoso by chtěla najít nenáročné a uživatelsky přívětivý způsob, jak určit demografické údaje zákazníků, věrnostní věci a reakce na ukládání a zobrazování produktů.

## <a name="solution"></a>Řešení

Tento vzor maloobchodní analýzy používá vrstvený přístup k Inferencing na hranici. Pomocí Custom Vision AI dev Kit se k analýze odesílají jenom obrázky s lidskými obličejemi do privátního centra Azure Stack, na kterém běží Azure Cognitive Services. Anonymní agregovaná data se do Azure odešlou pro agregaci napříč všemi obchody a vizualizací v Power BI. Kombinování hraničního a veřejného cloudu umožňuje společnosti Contoso využít moderní technologii AI a zároveň zůstat v souladu se svými podnikovými zásadami a respektující soukromí svých zákazníků.

[![Řešení Pattern Footfall Detection](media/pattern-retail-footfall-detection/solution-architecture.png)](media/pattern-retail-footfall-detection/solution-architecture.png)

Tady je přehled toho, jak řešení funguje:

1. Sada Custom Vision AI dev Kit Získá konfiguraci z IoT Hub, která nainstaluje IoT Edge modul runtime a model ML.
2. Pokud se model dohlíží na osobu, převezme obrázek a nahraje ho pro Azure Stack úložiště objektů BLOB hub.
3. Služba BLOB Service aktivuje funkci Azure ve službě Azure Stack hub.
4. Funkce Azure volá kontejner s rozhraní API pro rozpoznávání tváře, aby získala data demografických a emoce z obrázku.
5. Data se v clusteru Azure Event Hubs neodesílají a odešlou.
6. Cluster Event Hubs zasune data do Stream Analytics.
7. Stream Analytics agreguje data a odešle je do Power BI.

## <a name="components"></a>Komponenty

Toto řešení používá následující komponenty:

| Vrstva | Komponenta | Popis |
|----------|-----------|-------------|
| Hardware v obchodě | [Custom Vision AI dev Kit](https://azure.github.io/Vision-AI-DevKit-Pages/) | Poskytuje filtrování in-Store pomocí místního modelu ML, který zachycuje jenom obrázky lidí pro účely analýzy. Bezpečně zřízené a aktualizované prostřednictvím IoT Hub.<br><br>|
| Azure | [Azure Event Hubs](/azure/event-hubs/) | Azure Event Hubs poskytuje škálovatelnou platformu pro ingestování anonymních dat, která se v Azure Stream Analytics integrují s využitím. |
|  | [Azure Stream Analytics](/azure/stream-analytics/) | Úloha Azure Stream Analytics agreguje data a seskupuje je do 15 sekund Windows pro vizualizaci. |
|  | [Microsoft Power BI](https://powerbi.microsoft.com/) | Power BI poskytuje snadno použitelné rozhraní řídicího panelu pro zobrazení výstupu z Azure Stream Analytics. |
| Azure Stack Hub | [App Service](/azure-stack/operator/azure-stack-app-service-overview) | Poskytovatel prostředků App Service (RP) poskytuje základ pro komponenty Edge, včetně funkcí hostování a správy pro webové aplikace/rozhraní API a funkce. |
| | Cluster [modulu AKS (](https://github.com/Azure/aks-engine) Azure Kubernetes Service) | AKS RP s AKS-Engine clusteru nasazeným do centra Azure Stack poskytuje škálovatelný a odolný modul pro spuštění kontejneru rozhraní API pro rozpoznávání tváře. |
| | [Kontejnery rozhraní API pro rozpoznávání tváře](/azure/cognitive-services/face/face-how-to-install-containers) Azure Cognitive Services| Azure Cognitive Services RP s rozhraní API pro rozpoznávání tváře Containers poskytuje demografické, emoce a jedinečné zjišťování návštěvníků v privátní síti společnosti Contoso. |
| | Blob Storage | Image zachycené ze sady AI dev Kit se nahrají do úložiště objektů BLOB v centru Azure Stack. |
| | Azure Functions | Funkce Azure spuštěná v centru Azure Stack přijímá vstup z úložiště objektů BLOB a spravuje interakce s rozhraní API pro rozpoznávání tváře. Emituje data do clusteru Event Hubs umístěného v Azure.<br><br>|

## <a name="issues-and-considerations"></a>Problémy a důležité informace

Při rozhodování, jak implementovat toto řešení, vezměte v úvahu následující body:

### <a name="scalability"></a>Škálovatelnost

Pokud chcete toto řešení povolit pro škálování napříč několika fotoaparáty a umístěními, musíte zajistit, aby všechny součásti mohly zpracovat zvýšené zatížení. Možná budete muset provést následující akce:

- Zvyšte počet Stream Analytics jednotek streamování.
- Horizontální navýšení kapacity rozhraní API pro rozpoznávání tváře nasazení.
- Zvyšte propustnost Event Hubs clusteru.
- V extrémních případech může být potřeba migrovat z Azure Functions do virtuálního počítače.

### <a name="availability"></a>Dostupnost

Vzhledem k tomu, že toto řešení je vrstveno, je důležité vzít v úvahu, jak řešit potíže se sítí nebo výpadky napájení. V závislosti na obchodních potřebách můžete chtít implementovat mechanismus pro místní ukládání imagí do mezipaměti a pak předána do centra Azure Stack, když se připojení vrátí. Pokud je umístění dostatečně velké, může být lepší volbou nasazení Data Box Edge s kontejnerem rozhraní API pro rozpoznávání tváře do tohoto umístění.

### <a name="manageability"></a>Možnosti správy

Toto řešení může zahrnovat mnoho zařízení a umístění, což by mohlo mít nepraktický. [Služby IoT v Azure](/azure/iot-fundamentals/) je možné použít k automatickému uvedení nových umístění a zařízení do režimu online a jejich udržování v aktuálním stavu.

### <a name="security"></a>Zabezpečení

Toto řešení zachycuje obrázky zákazníků a má jistotu na nejdůležitější aspekty. Zajistěte, aby všechny účty úložiště byly zabezpečené se správnými zásadami přístupu a pravidelně střídat klíče. Ujistěte se, že účty úložiště a Event Hubs mají zásady uchovávání informací, které splňují předpisy pro ochranu osobních údajů podniku a vlády Také nezapomeňte nastavit úroveň přístupu uživatele. Vrstvení zajišťuje, že uživatelé mají přístup jenom k datům, která potřebují pro jejich roli.

## <a name="next-steps"></a>Další kroky

Další informace o tématech představených v tomto článku:

- Podívejte se na [model vrstvené dat](https://aka.ms/tiereddatadeploy), který je vydaný vzorem detekce Footfall.
- Další informace o používání vlastní vize najdete v sadě [Custom Vision AI dev Kit](https://azure.github.io/Vision-AI-DevKit-Pages/) . 

Až budete připraveni otestovat příklad řešení, pokračujte v [Průvodci nasazením Footfall Detection](solution-deployment-guide-retail-footfall-detection.md). Průvodce nasazením poskytuje podrobné pokyny pro nasazení a testování jeho komponent.
