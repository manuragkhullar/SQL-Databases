# SQL-Databases


WITH 
    skus AS (
            SELECT sku.skuid, sku.skudefaultname, cat.categoryname, br.brandname, sku.retailerlink, itemsku.itemid,con.countryname,sup.suppliername
            FROM `emi-lv-viads.ds0.sku` sku
                INNER JOIN `emi-lv-viads.ds0.country` con ON sku.countryid = con.countryid
                INNER JOIN `emi-lv-viads.ds0.category` cat ON sku.categoryid = cat.categoryid
                INNER JOIN `emi-lv-viads.ds0.brand` br ON sku.brandid = br.brandid AND sku.countryid=br.countryid
                INNER JOIN `emi-lv-viads.ds0.itemsku` itemsku ON itemsku.skuid = sku.skuid
                INNER JOIN `emi-lv-viads.ds0.retailer`ret ON ret.retailerid = sku.retailerid
                INNER JOIN `emi-lv-viads.ds0.supplier`sup ON sup.supplierid = br.supplierid
        WHERE con.regionname IN ('Eastern Europe','Western Europe')
        and sup.suppliername IN ("Nestlé SA","Procter & Gamble Co, The","Unilever Group","L'Oréal Groupe","Coca-Cola Co, The","Coca-Cola Amatil Ltd","Coca-Cola Hellenic Bottling Co SA","Kraft Heinz Co","Heinz","Danone Universal Robina Beverages","Danone, Groupe","PepsiCo Inc","Mars Inc","Kimberly-Clark Corp","Colgate-Palmolive Co","Beiersdorf AG","Mondelez International Inc","Mondelez Hungária Kft","General Mills","General Mills Inc","Johnson & Johnson Inc")
            AND itemsku.isInVia=1
        ),
    features AS (
        SELECT skus.skuid, fet.featurename, infd.featurevalue
        FROM skus
        INNER JOIN `emi-lv-viads.ds0.itemnumericfeaturedata` infd ON skus.itemid = infd.itemid
        INNER JOIN `emi-lv-viads.ds0.feature` fet ON infd.featureid = fet.featureid
        INNER JOIN `emi-lv-viads.ds0.unit` unt ON infd.unitid = unt.unitid
        WHERE infd.featureid IN (1,2,3)
    ),
    p_features AS (
        SELECT skus.skuid,
            MAX(IF(features.featurename LIKE '%Weight', features.featurevalue, null)) Weight,
            MAX(IF(features.featurename LIKE '%Volume', features.featurevalue, null)) Volume,
            MAX(IF(features.featurename LIKE '%Quantity', features.featurevalue, null)) Quantity,
        FROM skus
        INNER JOIN features ON features.skuid = skus.skuid
        GROUP BY skus.skuid
        ),
    period_skus AS (
        SELECT skus.*, fea.featurename, txtval.textvalue
        FROM skus
            INNER JOIN `emi-lv-viads.ds0.itemtextualfeaturedata` itfd ON skus.itemid = itfd.itemid
            INNER JOIN `emi-lv-viads.ds0.textvalue` txtval ON txtval.textvalueid = itfd.textvalueid
            INNER JOIN `emi-lv-viads.ds0.feature` fea ON fea.featureid = itfd.featureid
        WHERE fea.featurename IN ('Functional Benefit/Claim', 'Health Claims', 'Free From','No Artificial Ingredients','No Chemical Additives','Sustainable Packaging','Sustainable Sourcing','Diets','Pack Type','Flavour')
    ),
    e_features AS (
        SELECT skus.skuid,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Functional Benefit/Claim', period_skus.textvalue, null)) Functional_Benefit_Claims,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Health Claims', period_skus.textvalue, null)) Health_Claims,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Free From', period_skus.textvalue, null)) Free_From,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'No Artificial Ingredients', period_skus.textvalue, null)) No_Artificial_Ingredients,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'No Chemical Additives', period_skus.textvalue, null)) No_Chemical_Additives,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Sustainable Packaging', period_skus.textvalue, null)) Sustainable_Packaging,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Sustainable Sourcing', period_skus.textvalue, null)) Sustainable_Sourcing,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Diets', period_skus.textvalue, null)) Diets,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Pack Type', period_skus.textvalue, null)) Pack_Type,
            STRING_AGG(distinct IF(period_skus.featurename LIKE 'Flavour', period_skus.textvalue, null)) Flavour
        FROM skus
        INNER JOIN period_skus ON period_skus.skuid = skus.skuid
        GROUP BY skus.skuid
        ),
    prices AS (
            SELECT skus.skuid, snfd.price, 
                EXTRACT(year FROM per.periodstartdate) AS year
            FROM skus
            INNER JOIN `emi-lv-viads.ds0.skunumericext` snfd ON skus.skuid = snfd.skuid
            INNER JOIN `emi-lv-viads.ds0.period` per ON snfd.periodid = per.periodid
            WHERE per.periodname BETWEEN '2021-01-01' AND '2021-12-31'
                AND per.periodtypeid = 1
            ),
    annual_prices AS (
            SELECT prices.skuid,prices.year,
                percentile_cont(prices.price,0.5) over (partition by skuid,year) as Price2021
            FROM prices
            )
SELECT DISTINCT * 
FROM skus
    LEFT JOIN p_features USING (skuid)
    LEFT JOIN e_features USING (skuid)
    INNER JOIN annual_prices USING (skuid)
ORDER BY skuid
