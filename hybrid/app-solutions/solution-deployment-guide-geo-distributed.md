---
title: Přímý provoz s geograficky distribuovanou aplikací pomocí Azure a centra Azure Stack
description: Naučte se směrovat provoz do konkrétních koncových bodů pomocí geograficky distribuované aplikace s využitím Azure a centra Azure Stack.
author: BryanLa
ms.topic: article
ms.date: 11/05/2019
ms.author: bryanla
ms.reviewer: anajod
ms.lastreviewed: 11/05/2019
ms.openlocfilehash: 8f2b7e48a62896acfce7293dcd4f18d5a43add01
ms.sourcegitcommit: bb3e40b210f86173568a47ba18c3cc50d4a40607
ms.translationtype: MT
ms.contentlocale: cs-CZ
ms.lasthandoff: 06/17/2020
ms.locfileid: "84910521"
---
# <a name="direct-traffic-with-a-geo-distributed-app-using-azure-and-azure-stack-hub"></a>Přímý provoz s geograficky distribuovanou aplikací pomocí Azure a centra Azure Stack

Naučte se směrovat provoz do konkrétních koncových bodů na základě různých metrik pomocí vzoru geograficky distribuovaných aplikací. Když vytvoříte profil Traffic Manager s využitím geografického směrování a konfigurace koncového bodu, zajistíte směrování informací na koncové body na základě regionálních požadavků, podnikových a mezinárodních předpisů a vašich datových potřeb.

V tomto řešení sestavíte ukázkové prostředí pro:

> [!div class="checklist"]
> - Vytvořte geograficky distribuovanou aplikaci.
> - Použijte Traffic Manager k zacílení aplikace.

## <a name="use-the-geo-distributed-apps-pattern"></a>Použití vzoru geograficky distribuovaných aplikací

Pomocí geograficky distribuovaného vzoru vaše aplikace zahrnuje oblasti. Můžete být ve výchozím nastavení veřejným cloudem, ale někteří uživatelé mohou vyžadovat, aby data zůstala ve své oblasti. Na základě svých požadavků můžete směrovat uživatele na nejvhodnější Cloud.

### <a name="issues-and-considerations"></a>Problémy a důležité informace

#### <a name="scalability-considerations"></a>Aspekty zabezpečení

Řešení, které v tomto článku sestavíte, se nevejde na škálovatelnost. Pokud se ale používá v kombinaci s jinými řešeními Azure a místními řešeními, můžete přizpůsobit požadavky na škálovatelnost. Informace o vytvoření hybridního řešení s automatickým škálováním prostřednictvím Traffic Manageru najdete v tématu [vytváření řešení pro škálování mezi cloudy pomocí Azure](solution-deployment-guide-cross-cloud-scaling.md).

#### <a name="availability-considerations"></a>Aspekty dostupnosti

Stejně jako v případě, že se jedná o požadavky na škálovatelnost, toto řešení přímo neřeší dostupnost. V rámci tohoto řešení je ale možné implementovat Azure a místní řešení, aby se zajistila vysoká dostupnost všech součástí, které se týkají.

### <a name="when-to-use-this-pattern"></a>Kdy se má tento model použít

- Vaše organizace má mezinárodní pobočky vyžadující vlastní regionální zásady zabezpečení a distribuce.

- Každá z poboček vaší organizace si vyžádá data o zaměstnancích, firmách a obchodních přístavech, což vyžaduje, aby se aktivita vytváření sestav na místní a časová pásma.

- Vysoce škálovatelné požadavky jsou splněné horizontálním škálováním aplikací s více nasazeními aplikací v jedné oblasti a mezi oblastmi, které zpracovávají extrémní požadavky na zatížení.

### <a name="planning-the-topology"></a>Plánování topologie

Před vytvořením kapacity distribuovaných aplikací vám pomůže tyto věci:

- **Vlastní doména pro aplikaci:** Jaký je vlastní název domény, který budou zákazníci používat pro přístup k aplikaci? Pro ukázkovou aplikaci je vlastní název domény *www \. scalableasedemo.com.*

- **Doména Traffic Manager:** Při vytváření [profilu Traffic Manager Azure](https://docs.microsoft.com/azure/traffic-manager/traffic-manager-manage-profiles)se vybere název domény. Tento název se používá v kombinaci s příponou *trafficmanager.NET* k registraci položky domény spravované pomocí Traffic Manager. Pro ukázkovou aplikaci je zvolený název *škálovatelný-pomocný-demo*. Výsledkem je, že úplný název domény, který je spravovaný nástrojem Traffic Manager, je *Scalable-ASE-demo.trafficmanager.NET*.

- **Strategie škálování aplikace:** Rozhodněte se, jestli budou nároky na aplikace distribuované napříč několika App Service prostředími v jedné oblasti, několika oblastech nebo kombinací obou přístupů. Rozhodnutí by mělo být založené na očekáváních, kde se bude nacházet na provozu zákazníků, a na tom, jak se může dál škálovat i zbytek podpory back-endové infrastruktury aplikace. Například s bezstavovou aplikací 100% se dá aplikace hromadně škálovat s využitím kombinace více App Service prostředí na oblast Azure vynásobená App Service prostředími nasazenými napříč několika oblastmi Azure. Díky více než 15 globálním oblastem Azure, ze kterých si můžete vybrat, můžou zákazníci skutečně vytvořit vysoce škálovatelné nároky na aplikace na úrovni Hyper. Pro ukázkovou aplikaci, která se tady používá, se v jedné oblasti Azure (Střed USA – jih) vytvořila tři App Service prostředí.

- **Konvence pojmenování pro prostředí App Service:** Každé App Service prostředí vyžaduje jedinečný název. Mimo jedno nebo dvě App Service prostředí je vhodné, abyste měli zásady vytváření názvů, které vám pomůžou identifikovat každé App Service prostředí. Pro ukázkovou aplikaci, která se tady používá, se použila jednoduchá konvence pojmenování. Názvy tří App Service prostředí jsou *fe1ase*, *fe2ase*a *fe3ase*.

- **Konvence pojmenování pro aplikace:** Vzhledem k tomu, že bude nasazeno několik instancí aplikace, je pro každou instanci nasazené aplikace nutné zadat název. S App Service Environment pro Power Apps je možné použít stejný název aplikace i v různých prostředích. Vzhledem k tomu, že každé App Service prostředí má jedinečnou příponu domény, mohou vývojáři zvolit, aby v každém prostředí znovu převedly stejný název aplikace. Například vývojář může mít aplikace, které jsou pojmenovány takto: *MyApp.foo1.p.azurewebsites.NET*, *MyApp.foo2.p.azurewebsites.NET*, *MyApp.foo3.p.azurewebsites.NET*a tak dále. Pro aplikaci, která se tady používá, má každá instance aplikace jedinečný název. Používané názvy instancí aplikace jsou *webfrontend1*, *webfrontend2*a *webfrontend3*.

> [!Tip]  
> ![hybrid-pillars.png](./media/solution-deployment-guide-cross-cloud-scaling/hybrid-pillars.png)  
> Centrum Microsoft Azure Stack je rozšířením Azure. Centrum Azure Stack přináší flexibilitu a inovace cloud computingu do místního prostředí. tím se umožní jenom hybridní cloud, který umožňuje vytvářet a nasazovat hybridní aplikace odkudkoli.  
> 
> Články [týkající se návrhu hybridní aplikace](overview-app-design-considerations.md) prověří pilíře kvality softwaru (umístění, škálovatelnost, dostupnost, odolnost, možnosti správy a zabezpečení) pro navrhování, nasazování a provozování hybridních aplikací. Pokyny k návrhu pomáhají při optimalizaci návrhu hybridní aplikace a minimalizaci výzev v produkčních prostředích.

## <a name="part-1-create-a-geo-distributed-app"></a>Část 1: vytvoření geografické distribuované aplikace

V této části vytvoříte webovou aplikaci.

> [!div class="checklist"]
> - Vytváření webových aplikací a publikování.
> - Přidejte kód pro Azure Repos.
> - Najeďte na sestavení aplikace na více cloudových cílů.
> - Správa a konfigurace procesu CD

### <a name="prerequisites"></a>Požadavky

Vyžaduje se instalace předplatného Azure a centra Azure Stack.

### <a name="geo-distributed-app-steps"></a>Postup geografické distribuované aplikace

### <a name="obtain-a-custom-domain-and-configure-dns"></a>Získání vlastní domény a konfigurace DNS

Aktualizujte soubor zóny DNS pro doménu. Azure AD pak může ověřit vlastnictví vlastního názvu domény. Použijte [Azure DNS](https://docs.microsoft.com/azure/dns/dns-getstarted-portal) pro Azure/externí záznamy DNS v Azure, nebo přidejte položku DNS v [jiném registrátoru DNS](https://support.office.com/article/Create-DNS-records-for-Office-365-when-you-manage-your-DNS-records-b0f3fdca-8a80-4e8e-9ef3-61e8a2a9ab23/).

1. Zaregistrujte vlastní doménu s veřejným registrátorem.

2. Přihlaste se k registrátorovi názvu domény. K provedení aktualizací DNS může být nutný schválený správce.

3. Aktualizujte soubor zóny DNS pro doménu tak, že přidáte položku DNS, kterou poskytuje Azure AD. Položka DNS nemění chování, například směrování pošty nebo hostování webu.

### <a name="create-web-apps-and-publish"></a>Vytváření webových aplikací a publikování

Nastavte hybridní průběžnou integraci/průběžné doručování (CI/CD), abyste nasadili webovou aplikaci do Azure a centra Azure Stack a automaticky nastavili změny do obou cloudů.

> [!Note]  
> Azure Stack centrum se správnými obrázky publikovanými pro spuštění (Windows Server a SQL) a vyžaduje se nasazení App Service. Další informace najdete v tématu [předpoklady pro nasazení App Service v centru Azure Stack](/azure-stack/operator/azure-stack-app-service-before-you-get-started.md).

#### <a name="add-code-to-azure-repos"></a>Přidat kód pro Azure Repos

1. Přihlaste se k aplikaci Visual Studio pomocí **účtu s právy pro vytváření projektu** na Azure Repos.

    CI/CD se může vztahovat jak na kód aplikace, tak pro kód infrastruktury. Použijte [šablony Azure Resource Manager](https://azure.microsoft.com/resources/templates/) pro vývoj privátního i hostovaného cloudu.

    ![Připojení k projektu v aplikaci Visual Studio](media/solution-deployment-guide-geo-distributed/image1.JPG)

2. **Naklonujte úložiště** vytvořením a otevřením výchozí webové aplikace.

    ![Klonovat úložiště v aplikaci Visual Studio](media/solution-deployment-guide-geo-distributed/image2.png)

### <a name="create-web-app-deployment-in-both-clouds"></a>Vytvoření nasazení webové aplikace v obou cloudech

1. Upravte soubor **WebApplication. csproj** : vyberte `Runtimeidentifier` a přidejte `win10-x64` . (Viz dokumentace k [samoobslužnému nasazení](https://docs.microsoft.com/dotnet/core/deploying/deploy-with-vs#simpleSelf) .)

    ![Upravit soubor projektu webové aplikace v aplikaci Visual Studio](media/solution-deployment-guide-geo-distributed/image3.png)

2. **Vrácením kódu se změnami Azure Repos** používání Team Explorer.

3. Potvrďte, že **kód aplikace** byl zkontrolován do Azure Repos.

### <a name="create-the-build-definition"></a>Vytvoření definice sestavení

1. **Přihlaste se k Azure Pipelines** a potvrďte schopnost vytvářet definice sestavení.

2. Přidejte `-r win10-x64` kód. Tento dodatek je nezbytný pro aktivaci samostatného nasazení pomocí .NET Core.

    ![Přidejte kód do definice sestavení v Azure Pipelines](media/solution-deployment-guide-geo-distributed/image4.png)

3. **Spusťte sestavení**. Proces [sestavení samostatného nasazení](https://docs.microsoft.com/dotnet/core/deploying/deploy-with-vs#simpleSelf) bude publikovat artefakty, které se dají spouštět v Azure a centra Azure Stack.

#### <a name="using-an-azure-hosted-agent"></a>Použití hostovaného agenta Azure

Použití hostovaného agenta v Azure Pipelines je pohodlnou možností pro sestavování a nasazování webových aplikací. Údržba a upgrady se automaticky provádí Microsoft Azure, což umožňuje nepřerušované nasazení, testování a nasazování.

### <a name="manage-and-configure-the-cd-process"></a>Správa a konfigurace procesu CD

Azure DevOps Services poskytují vysoce konfigurovatelný a spravovatelný kanál pro vydání do více prostředí, jako jsou vývojové, pracovní, QA a produkční prostředí. zahrnutí požadavku na schválení v určitých fázích.

## <a name="create-release-definition"></a>Vytvořit definici vydané verze

1. Kliknutím na tlačítko **plus** přidejte novou verzi na kartě **vydání** v části **sestavení a vydání** Azure DevOps Services.

    ![Vytvoření definice verze v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image5.png)

2. Použijte šablonu nasazení Azure App Service.

   ![Použít šablonu nasazení Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image6.png)

3. V části **Přidat artefakt**přidejte artefakt pro aplikaci Azure Cloud Build.

   ![Přidání artefaktu do cloudového sestavení Azure v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image7.png)

4. Na kartě kanál vyberte **fáze,** odkaz na úlohu prostředí a nastavte hodnoty cloudového prostředí Azure.

   ![Nastavení hodnot cloudového prostředí Azure v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image8.png)

5. Nastavte **Název prostředí** a vyberte **předplatné Azure** pro koncový bod cloudu Azure.

      ![Vyberte předplatné Azure pro koncový bod cloudu Azure v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image9.png)

6. V části **název služby App Service**nastavte požadovaný název služby Azure App Service.

      ![Nastavte název služby Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image10.png)

7. Do pole **fronta agenta** pro hostované cloudové prostředí Azure zadejte "hostované VS2017".

      ![Nastavte frontu agentů pro prostředí hostované v cloudu Azure v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image11.png)

8. V nabídce nasadit Azure App Service vyberte pro prostředí platný **balíček nebo složku** . Vyberte **OK** do **umístění složky**.
  
      ![Vyberte balíček nebo složku pro prostředí Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image12.png)

      ![Vyberte balíček nebo složku pro prostředí Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image13.png)

9. Uložte všechny změny a vraťte se do **kanálu uvolnění**.

    ![Uložení změn v kanálu vydání v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image14.png)

10. Přidejte nový artefakt, který vybírá sestavení pro aplikaci Azure Stack hub.

    ![Přidání nového artefaktu pro aplikaci Azure Stackového centra v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image15.png)


11. Přidejte další prostředí pomocí Azure App Service nasazení.

    ![Přidání prostředí do nasazení Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image16.png)

12. Pojmenujte nové prostředí Azure Stack hub.

    ![Název prostředí při nasazení Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image17.png)

13. Na kartě **úloha** najděte prostředí Azure Stack hub.

    ![Azure Stack prostředí centra v Azure DevOps Services v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image18.png)

14. Vyberte předplatné pro koncový bod centra Azure Stack.

    ![Vyberte předplatné pro koncový bod centra Azure Stack v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image19.png)

15. Jako název služby App Service nastavte název webové aplikace centra Azure Stack.

    ![Nastavit název webové aplikace Azure Stack hub v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image20.png)

16. Vyberte agenta centra Azure Stack.

    ![Vybrat agenta centra Azure Stack v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image21.png)

17. V části nasadit Azure App Service vyberte platný **balíček nebo složku** pro prostředí. Vyberte **OK** do umístění složky.

    ![Vyberte složku pro nasazení Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image22.png)

    ![Vyberte složku pro nasazení Azure App Service v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image23.png)

18. V části karta proměnné přidejte proměnnou s názvem `VSTS\_ARM\_REST\_IGNORE\_SSL\_ERRORS` , nastavte její hodnotu na **true**a obor na Azure Stack hub.

    ![Přidání proměnné do nasazení aplikace Azure v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image24.png)

19. V obou artefaktech vyberte ikonu triggeru **průběžného** nasazování a povolte aktivační událost **pokračování** nasazení.

    ![Vyberte aktivační událost průběžného nasazování v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image25.png)

20. Vyberte ikonu podmínky **před nasazením** v prostředí Azure Stack hub a nastavte Trigger na **po vydání.**

    ![Vyberte podmínky předběžného nasazení v Azure DevOps Services](media/solution-deployment-guide-geo-distributed/image26.png)

21. Uložte všechny změny.

> [!Note]  
> Některá nastavení pro úlohy mohla být při vytváření definice verze ze šablony automaticky definována jako [proměnné prostředí](https://docs.microsoft.com/azure/devops/pipelines/release/variables?view=vsts&tabs=batch#custom-variables) . Tato nastavení se nedají upravit v nastavení úlohy. místo toho je nutné vybrat nadřazenou položku prostředí pro úpravu těchto nastavení.

## <a name="part-2-update-web-app-options"></a>Část 2: aktualizace možností webové aplikace

[Azure App Service ](https://docs.microsoft.com/azure/app-service/overview) je vysoce škálovatelná služba s automatickými opravami pro hostování webů.

![Azure App Service](media/solution-deployment-guide-geo-distributed/image27.png)

> [!div class="checklist"]
> - Mapování existujícího vlastního názvu DNS na Azure Web Apps.
> - Použijte **záznam CNAME** a **záznam** a k mapování vlastního názvu DNS na App Service.

### <a name="map-an-existing-custom-dns-name-to-azure-web-apps"></a>Mapování existujícího vlastního názvu DNS na Azure Web Apps

> [!Note]  
> Použijte záznam CNAME pro všechny vlastní názvy DNS kromě kořenové domény (například northwind.com).

Pokud chcete do služby App Service migrovat živý web a jeho název domény DNS, přečtěte si téma [Migrace aktivního názvu DNS do služby Azure App Service](https://docs.microsoft.com/azure/app-service/manage-custom-dns-migrate-domain).

### <a name="prerequisites"></a>Požadavky

Dokončení tohoto řešení:

- [Vytvořte aplikaci App Service](https://docs.microsoft.com/azure/app-service/)nebo použijte aplikaci vytvořenou pro jiné řešení.

- Zakupte název domény a zajistěte přístup k registru DNS pro poskytovatele domény.

Aktualizujte soubor zóny DNS pro doménu. Azure AD ověří vlastnictví vlastního názvu domény. Použijte [Azure DNS](https://docs.microsoft.com/azure/dns/dns-getstarted-portal) pro Azure/externí záznamy DNS v Azure, nebo přidejte položku DNS v [jiném registrátoru DNS](https://support.office.com/article/Create-DNS-records-for-Office-365-when-you-manage-your-DNS-records-b0f3fdca-8a80-4e8e-9ef3-61e8a2a9ab23/).

- Zaregistrujte vlastní doménu s veřejným registrátorem.

- Přihlaste se k registrátorovi názvu domény. (K provedení aktualizací DNS může být nutný schválený správce.)

- Aktualizujte soubor zóny DNS pro doménu tak, že přidáte položku DNS, kterou poskytuje Azure AD.

Pokud například chcete přidat položky DNS pro northwindcloud.com a webové \. northwindcloud.com, nakonfigurujte nastavení DNS pro kořenovou doménu northwindcloud.com.

> [!Note]  
> Název domény může být zakoupen pomocí [Azure Portal](https://docs.microsoft.com/azure/app-service/manage-custom-dns-buy-domain). Abyste mohli mapovat vlastní název DNS na webovou aplikaci, [plán služby App Service](https://azure.microsoft.com/pricing/details/app-service/) příslušné webové aplikace musí být na placené úrovni (**Shared**, **Basic**, **Standard** nebo **Premium**).

### <a name="create-and-map-cname-and-a-records"></a>Vytvoření a mapování záznamů CNAME a a

#### <a name="access-dns-records-with-domain-provider"></a>Přístup k záznamům DNS u poskytovatele domény

> [!Note]  
>  Pomocí Azure DNS můžete nakonfigurovat vlastní název DNS pro Azure Web Apps. Další informace najdete v tématu popisujícím [použití Azure DNS k určení nastavení vlastní domény pro službu Azure](https://docs.microsoft.com/azure/dns/dns-custom-domain).

1. Přihlaste se na web hlavního poskytovatele.

2. Vyhledejte stránku pro správu záznamů DNS. Každý poskytovatel domény má vlastní rozhraní záznamů DNS. Hledejte oblasti webu označené jako **Domain Name** (Název domény), **DNS** nebo **Name Server Management** (Správa názvových serverů).

Stránky záznamů DNS je možné zobrazit v **mých doménách**. Vyhledejte soubor s názvem **zóny**, **záznamy DNS**nebo **pokročilou konfiguraci**.

Následující snímek obrazovky obsahuje příklad stránky záznamů DNS:

![Příklad stránky záznamů DNS](media/solution-deployment-guide-geo-distributed/image28.png)

1. V registrátoru názvu domény vyberte **Přidat nebo vytvořit** a vytvořte záznam. Někteří poskytovatelé nabízejí různé odkazy pro přidání různých typů záznamů. Projděte si dokumentaci k poskytovateli.

2. Přidejte záznam CNAME pro mapování subdomény na výchozí název hostitele aplikace.

   V \. příkladech domény northwindcloud.com www přidejte záznam CNAME, na který se název mapuje `<app_name>.azurewebsites.net` .

Po přidání CNAME bude stránka záznamů DNS vypadat jako v následujícím příkladu:

![Přechod do aplikace Azure na portálu](media/solution-deployment-guide-geo-distributed/image29.png)

### <a name="enable-the-cname-record-mapping-in-azure"></a>Povolení mapování záznamu CNAME v Azure

1. Na nové kartě se přihlaste k Azure Portal.

2. Přejděte na App Services.

3. Vyberte webová aplikace.

4. V levém navigačním panelu na stránce aplikace na webu Azure Portal vyberte **Vlastní domény**.

5. Vyberte **+** ikonu vedle **Přidat název hostitele**.

6. Zadejte plně kvalifikovaný název domény, třeba `www.northwindcloud.com` .

7. Vyberte **Ověřit**.

8. Pokud je uvedeno jinak, přidejte další záznamy jiných typů ( `A` nebo `TXT` ) do záznamů DNS registrátora názvu domény. Azure nabídne hodnoty a typy těchto záznamů:

   a.  Záznam **A** pro mapování na IP adresu aplikace.

   b.  Záznam **TXT** pro mapování na výchozí název hostitele aplikace `<app_name>.azurewebsites.net`. App Service používá tento záznam pouze v době konfigurace k ověření vlastního vlastnictví domény. Po ověření odstraňte záznam TXT.

9. Dokončete tuto úlohu na kartě registrátor domény a znovu ověřte, dokud není aktivováno tlačítko **Přidat název hostitele** .

10. Ujistěte se, že **typ záznamu názvu hostitele** je nastavený na **CNAME** (www.example.com nebo libovolná subdoména).

11. Vyberte **Přidat název hostitele**.

12. Zadejte plně kvalifikovaný název domény, třeba `northwindcloud.com` .

13. Vyberte **Ověřit**. Je aktivováno **Přidání** .

14. Ujistěte se, že **typ záznamu názvu hostitele** je nastavený na **záznam** (example.com).

15. **Přidejte název hostitele**.

    Může trvat nějakou dobu, než se nové názvy hostitelů projeví na stránce **vlastní domény** aplikace. Zkuste aktualizovat prohlížeč, aby se data aktualizovala.
  
    ![Vlastní domény](media/solution-deployment-guide-geo-distributed/image31.png) 
  
    Pokud dojde k chybě, zobrazí se v dolní části stránky oznámení o chybě ověření. ![Chyba ověřování domény](media/solution-deployment-guide-geo-distributed/image32.png)

> [!Note]  
>  Výše uvedené kroky se můžou opakovat, aby se namapovala doména se zástupnými znaky ( \* . northwindcloud.com). To umožňuje přidání jakýchkoli dalších subdomén do této služby App Service, aniž by bylo nutné pro každé z nich vytvořit samostatný záznam CNAME. Nakonfigurujte toto nastavení podle pokynů registrátora.

#### <a name="test-in-a-browser"></a>Testování v prohlížeči

Přejděte na názvy DNS, které jste nakonfigurovali dříve (například `northwindcloud.com` nebo `www.northwindcloud.com` ).

## <a name="part-3-bind-a-custom-ssl-cert"></a>Část 3: vazba vlastního certifikátu SSL

V této části budeme:

> [!div class="checklist"]
> - Navažte vlastní certifikát SSL na App Service.
> - Vyvynuťte pro aplikaci protokol HTTPS.
> - Automatizujte vazbu certifikátu SSL pomocí skriptů.

> [!Note]  
> V případě potřeby Získejte certifikát SSL zákazníka v Azure Portal a navažte ho k webové aplikaci. Další informace najdete v [kurzu App Servicech certifikátů](https://docs.microsoft.com/azure/app-service/web-sites-purchase-ssl-web-site).

### <a name="prerequisites"></a>Požadavky

Dokončení tohoto řešení:

- [Vytvořte aplikaci App Service.](https://docs.microsoft.com/azure/app-service/)
- [Namapujte vlastní název DNS na webovou aplikaci.](https://docs.microsoft.com/azure/app-service/app-service-web-tutorial-custom-domain)
- Získejte certifikát SSL od důvěryhodné certifikační autority a použijte klíč k podepsání žádosti.

### <a name="requirements-for-your-ssl-certificate"></a>Požadavky na certifikát SSL

Pokud chcete ve službě App Service použít certifikát, musí splňovat všechny následující požadavky:

- Podepsáno důvěryhodnou certifikační autoritou.

- Exportováno jako soubor PFX chráněný heslem.

- Obsahuje privátní klíč minimálně 2048 bitů dlouhého.

- Obsahuje všechny zprostředkující certifikáty v řetězu certifikátů.

> [!Note]  
> **Certifikáty ECC (eliptický Curve Cryptography)** fungují s App Service, ale nejsou součástí této příručky. Pomoc při vytváření certifikátů ECC získáte od certifikační autority.

#### <a name="prepare-the-web-app"></a>Příprava webové aplikace

Aby bylo možné vytvořit navázání vlastního certifikátu SSL k webové aplikaci, musí být [plán App Service](https://azure.microsoft.com/pricing/details/app-service/) v úrovni **Basic**, **Standard**nebo **Premium** .

#### <a name="sign-in-to-azure"></a>Přihlášení k Azure

1. Otevřete [Azure Portal](https://portal.azure.com/) a pokračujte na webovou aplikaci.

2. V nabídce vlevo vyberte **App Services**a pak vyberte název webové aplikace.

![Vyberte možnost Webová aplikace v Azure Portal](media/solution-deployment-guide-geo-distributed/image33.png)

#### <a name="check-the-pricing-tier"></a>Kontrola cenové úrovně

1. V levé navigační části stránky webové aplikace se posuňte do části **Nastavení** a vyberte **škálovat nahoru (App Service plán)**.

    ![Škálování nabídky ve webové aplikaci](media/solution-deployment-guide-geo-distributed/image34.png)

1. Ujistěte se, že webová aplikace není na úrovni **Free** nebo **Shared** . Aktuální úroveň webové aplikace je zvýrazněna v tmavě modrém poli.

    ![Kontrolovat cenovou úroveň ve webové aplikaci](media/solution-deployment-guide-geo-distributed/image35.png)

Vlastní protokol SSL není podporován na úrovni **Free** nebo **Shared** . Pokud chcete provést škálování, postupujte podle kroků v následující části nebo na stránce **vyberte cenovou úroveň** a přejděte k části [odeslání a vytvoření vazby certifikátu SSL](https://docs.microsoft.com/azure/app-service/app-service-web-tutorial-custom-ssl).

#### <a name="scale-up-your-app-service-plan"></a>Vertikální navýšení kapacity plánu služby App Service

1. Vyberte některou z úrovní **Basic**, **Standard** nebo **Premium**.

2. Vyberte **Vybrat**.

![Výběr cenové úrovně pro vaši webovou aplikaci](media/solution-deployment-guide-geo-distributed/image36.png)

Operace škálování je dokončena, když se zobrazí oznámení.

![Oznámení vertikálního navýšení kapacity](media/solution-deployment-guide-geo-distributed/image37.png)

#### <a name="bind-your-ssl-certificate-and-merge-intermediate-certificates"></a>Vytvoření vazby certifikátu SSL a sloučení zprostředkujících certifikátů

Sloučí více certifikátů v řetězu.

1. **Otevřete každý certifikát** , který jste obdrželi v textovém editoru.

2. Vytvořte soubor pro sloučený certifikát nazvaný *mergedcertificate. CRT*. V textovém editoru zkopírujte do tohoto souboru obsah jednotlivých certifikátů. Pořadí certifikátů by mělo odpovídat jejich pořadí v řetězu certifikátů, počínaje vaším certifikátem a konče kořenovým certifikátem. Soubor bude vypadat jako v následujícím příkladu:

    ```Text

    -----BEGIN CERTIFICATE-----

    <your entire Base64 encoded SSL certificate>

    -----END CERTIFICATE-----

    -----BEGIN CERTIFICATE-----

    <The entire Base64 encoded intermediate certificate 1>

    -----END CERTIFICATE-----

    -----BEGIN CERTIFICATE-----

    <The entire Base64 encoded intermediate certificate 2>

    -----END CERTIFICATE-----

    -----BEGIN CERTIFICATE-----

    <The entire Base64 encoded root certificate>

    -----END CERTIFICATE-----
    ```

#### <a name="export-certificate-to-pfx"></a>Export certifikátu do formátu PFX

Exportujte sloučený certifikát SSL s privátním klíčem vygenerovaným certifikátem.

Soubor privátního klíče se vytvoří prostřednictvím OpenSSL. Pokud chcete certifikát exportovat do souboru PFX, spusťte následující příkaz a nahraďte zástupné symboly `<private-key-file>` a `<merged-certificate-file>` cestu k privátnímu klíči a souborem sloučeného certifikátu:

```powershell
openssl pkcs12 -export -out myserver.pfx -inkey <private-key-file> -in <merged-certificate-file>
```

Po zobrazení výzvy definujte heslo pro export pro nahrání certifikátu SSL pro App Service později.

Když se k vygenerování žádosti o certifikát používá služba IIS nebo **Certreq.exe** , nainstalujte certifikát do místního počítače a pak [certifikát EXPORTUJTE do souboru PFX](https://technet.microsoft.com/library/cc754329(v=ws.11).aspx).

#### <a name="upload-the-ssl-certificate"></a>Nahrajte certifikát SSL.

1. V levém navigačním panelu webové aplikace vyberte **Nastavení SSL** .

2. Vyberte **Odeslat certifikát**.

3. V **souboru certifikátu PFX**vyberte soubor PFX.

4. Do pole **heslo certifikátu**zadejte heslo vytvořené při exportování souboru PFX.

5. Vyberte **Nahrát**.

    ![Odeslat certifikát SSL](media/solution-deployment-guide-geo-distributed/image38.png)

Až App Service dokončí nahrávání certifikátu, zobrazí se na stránce **Nastavení SSL** .

![Nastavení protokolu SSL](media/solution-deployment-guide-geo-distributed/image39.png)

#### <a name="bind-your-ssl-certificate"></a>Vytvoření vazby certifikátu SSL

1. V části **vazby SSL** vyberte **Přidat vazbu**.

    > [!Note]  
    >  Pokud se certifikát nahrál, ale v rozevíracím seznamu název **hostitele** se nezobrazí v části názvy domén, zkuste aktualizovat stránku prohlížeče.

2. Na stránce **Přidat vazbu SSL** vyberte rozevírací seznam a vyberte název domény, který chcete zabezpečit, a certifikát, který chcete použít.

3. V části **Typ SSL** vyberte, jestli se má použít SSL na základě [**Indikace názvu serveru (SNI)**](https://en.wikipedia.org/wiki/Server_Name_Indication) nebo IP adresy.

    - **SSL založené na sni**: přidat se dá víc vazeb SSL založených na sni. Tato možnost umožňuje zabezpečení několika domén na stejné IP adrese pomocí několika certifikátů SSL. Většina moderních prohlížečů (včetně prohlížečů Internet Explorer, Chrome, Firefox a Opera) podporuje SNI (ucelenější informace o podpoře prohlížečů najdete v článku o [Indikaci názvu serveru](https://wikipedia.org/wiki/Server_Name_Indication)).

    - **SSL na základě IP adresy**: dá se přidat jenom jedna vazba SSL založená na IP adrese. Tato možnost umožňuje zabezpečení vyhrazené veřejné IP adresy pouze jedním certifikátem SSL. Chcete-li zabezpečit více domén, zabezpečte je pomocí stejného certifikátu SSL. Protokol SSL založený na protokolu IP je tradiční volbou pro vazby SSL.

4. Vyberte **Přidat vazbu**.

    ![Přidat vazbu SSL](media/solution-deployment-guide-geo-distributed/image40.png)

Až App Service dokončí nahrávání certifikátu, zobrazí se v oddílech **vazeb SSL** .

![Nahrávání vazeb SSL bylo dokončeno.](media/solution-deployment-guide-geo-distributed/image41.png)

#### <a name="remap-the-a-record-for-ip-ssl"></a>Přemapování záznamu A pro IP SSL

Pokud se ve webové aplikaci nepoužívá protokol SSL založený na protokolu IP, přeskočte na [test HTTPS pro vaši vlastní doménu](https://docs.microsoft.com/azure/app-service/app-service-web-tutorial-custom-ssl).

Ve výchozím nastavení webová aplikace používá sdílenou veřejnou IP adresu. Pokud je certifikát vázán pomocí protokolu SSL založeného na protokolu IP, App Service vytvoří novou a vyhrazenou IP adresu pro webovou aplikaci.

Pokud je záznam A mapován na webovou aplikaci, je nutné aktualizovat registr domény pomocí vyhrazené IP adresy.

Stránka **vlastní doména** je aktualizována novou vyhrazenou IP adresou. Zkopírujte tuto [IP adresu](https://docs.microsoft.com/azure/app-service/app-service-web-tutorial-custom-domain)a pak přemapujte [záznam a](https://docs.microsoft.com/azure/app-service/app-service-web-tutorial-custom-domain) na tuto novou IP adresu.

#### <a name="test-https"></a>Test HTTPS

V různých prohlížečích přejdete na adresu, `https://<your.custom.domain>` abyste zajistili, že se webová aplikace doručí.

![Přejít k webové aplikaci](media/solution-deployment-guide-geo-distributed/image42.png)

> [!Note]  
> Pokud dojde k chybám ověření certifikátu, může to být způsobeno certifikátem podepsaným svým držitelem nebo při exportu do souboru PFX byly pozměněny zprostředkující certifikáty.

#### <a name="enforce-https"></a>Vynucení protokolu HTTPS

Ve výchozím nastavení má kdokoli přístup k webové aplikaci přes HTTP. Je možné přesměrovat všechny požadavky HTTP na port HTTPS.

Na stránce webová aplikace vyberte **Nastavení SL**. Pak v části **Pouze HTTPS** vyberte **Zapnuto**.

![Vynucení protokolu HTTPS](media/solution-deployment-guide-geo-distributed/image43.png)

Po dokončení operace přejděte na libovolnou adresu URL protokolu HTTP, která odkazuje na aplikaci. Příklad:

- https://<app_name>. azurewebsites.net
- `https://northwindcloud.com`
- `https://www.northwindcloud.com`

#### <a name="enforce-tls-1112"></a>Vynucení protokolu TLS 1.1/1.2

Aplikace ve výchozím nastavení povolí protokol [TLS](https://wikipedia.org/wiki/Transport_Layer_Security) 1,0, který se už nepovažuje za zabezpečený oborovou normou (například [PCI DSS](https://wikipedia.org/wiki/Payment_Card_Industry_Data_Security_Standard)). Pokud chcete vynucovat vyšší verze protokolu TLS, postupujte následovně:

1. Na stránce webová aplikace v levém navigačním panelu vyberte **Nastavení SSL**.

2. V části **verze TLS**vyberte minimální verzi TLS.

    ![Vynucení protokolu TLS 1.1 nebo 1.2](media/solution-deployment-guide-geo-distributed/image44.png)

### <a name="create-a-traffic-manager-profile"></a>Vytvoření profilu Traffic Manageru

1. Vyberte **vytvořit prostředek**  >  **sítě**  >  **Traffic Manager profil**  >  **vytvořit**.

2. Část **Vytvořit profil služby Traffic Manager** vyplňte následovně:

    1. Do pole **název**zadejte název profilu. Tento název musí být jedinečný v rámci zóny přenosů manager.net a má za následek název DNS, trafficmanager.net, který se používá pro přístup k profilu Traffic Manager.

    2. V části **způsob směrování**vyberte **metodu geografického směrování**.

    3. V části **předplatné**vyberte předplatné, ve kterém chcete vytvořit tento profil.

    4. V části **Skupina prostředků** vytvořte novou skupinu prostředků, do které chcete profil umístit.

    5. V poli **Umístění skupiny prostředků** vyberte umístění skupiny prostředků. Toto nastavení odkazuje na umístění skupiny prostředků a nemá žádný vliv na globálně nasazený profil Traffic Manager.

    6. Vyberte **Vytvořit**.

    7. Po dokončení globálního nasazení profilu Traffic Manager se v příslušné skupině prostředků zobrazí jako jeden z prostředků.

        ![Skupiny prostředků v profilu Create Traffic Manager](media/solution-deployment-guide-geo-distributed/image45.png)

### <a name="add-traffic-manager-endpoints"></a>Přidání koncových bodů služby Traffic Manager

1. Na panelu hledání na portálu vyhledejte název **profilu Traffic Manager** vytvořeného v předchozí části a v zobrazených výsledcích vyberte profil Traffic Manageru.

2. V **Traffic Manager profil**v části **Nastavení** vyberte **koncové body**.

3. Vyberte možnost **Přidat**.

4. Přidává se koncový bod centra Azure Stack.

5. Jako **typ**vyberte **externí koncový bod**.

6. Zadejte **název** tohoto koncového bodu, v ideálním případě název centra Azure Stack.

7. Pro plně kvalifikovaný název domény (**FQDN**) použijte externí adresu URL pro webovou aplikaci Azure Stack hub.

8. V části geografické mapování vyberte oblast nebo kontinent, kde se prostředek nachází. Například **Evropa.**

9. V rozevíracím seznamu země/oblast, který se zobrazí, vyberte zemi, která se vztahuje k tomuto koncovému bodu. Například **Německo**.

10. Políčko **Přidat jako zakázaný** ponechte nezaškrtnuté.

11. Vyberte **OK**.

12. Přidání Koncový bod Azure:

    1. Jako **typ**vyberte **koncový bod Azure**.

    2. Zadejte **název** koncového bodu.

    3. Jako **typ cílového prostředku**vyberte **App Service**.

    4. V části **cílový prostředek**vyberte **možnost zvolit službu App Service** , abyste zobrazili výpis Web Apps v rámci stejného předplatného. V části **prostředek**vyberte službu App Service, která se používá jako první koncový bod.

13. V části geografické mapování vyberte oblast nebo kontinent, kde se prostředek nachází. Například **Severní Amerika/Střední Amerika/Karibská oblast.**

14. V rozevíracím seznamu země/oblast, který se zobrazí, ponechte toto políčko prázdné, pokud chcete vybrat všechna výše uvedená oblastní seskupení.

15. Políčko **Přidat jako zakázaný** ponechte nezaškrtnuté.

16. Vyberte **OK**.

    > [!Note]  
    >  Vytvořte alespoň jeden koncový bod s geografickým rozsahem všech (World), který bude sloužit jako výchozí koncový bod pro daný prostředek.

17. Když se dokončí přidávání obou koncových bodů, zobrazí se v **profilu Traffic Manager** spolu s jejich stavem monitorování jako **online**.

    ![Stav koncového bodu profilu Traffic Manager](media/solution-deployment-guide-geo-distributed/image46.png)

#### <a name="global-enterprise-relies-on-azure-geo-distribution-capabilities"></a>Globální podnik spoléhá na možnosti geografické distribuce Azure

Přímý přenos dat prostřednictvím Azure Traffic Manager a koncových bodů specifických pro geografie umožňuje globálním podnikům dodržovat regionální předpisy a zachovat předpisy a zajistit jejich kompatibilitu a zabezpečení, což je zásadní pro úspěch místních i vzdálených obchodních umístění.

## <a name="next-steps"></a>Další kroky

- Další informace o vzorech cloudu Azure najdete v tématu [vzory návrhu cloudu](https://docs.microsoft.com/azure/architecture/patterns).
