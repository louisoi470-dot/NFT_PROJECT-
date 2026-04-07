QUESTION 1.1

-- how much capacity is seaitng at every node and the utilization of that node
SELECT noo.node_type,
	sum(units_on_hand) as Avalible,
    sum(max_capacity) as capacity,
    cast((sum(units_on_hand) / sum(max_capacity)) * 100 as decimal(10, 2)) as 'capacity %',
    CAST(sum(daily_storage_cost_eur) as decimal(10,3)) as Total_holding_cost,
    cast((sum(daily_storage_cost_eur) / sum(units_on_hand)) as decimal (10,6)) as avg_holding_cost_per_item
FROM amazon_potfolio.inventory AS api
JOIN amazon_potfolio.nodes AS noo
    ON api.node_id = noo.node_id
group by noo.node_type

QUESTION 1.2 & 1.3

-- this is to see the volume of the different types of sku handled by the different nodes

WITH base AS (
    SELECT
        nd.node_type,

        SUM(CASE WHEN sk.velocity = 'fast' THEN iv.units_on_hand ELSE 0 END) AS fast_units,
        SUM(CASE WHEN sk.velocity = 'fast' THEN 1 ELSE 0 END) AS fast_units_deliv,
        SUM(CASE WHEN sk.velocity = 'medium' THEN iv.units_on_hand ELSE 0 END) AS medium_units,
        SUM(CASE WHEN sk.velocity = 'medium' THEN 1 ELSE 0 END) AS medium_units_deliv,
        SUM(CASE WHEN sk.velocity = 'slow' THEN iv.units_on_hand ELSE 0 END) AS slow_units,
        SUM(CASE WHEN sk.velocity = 'slow' THEN 1 ELSE 0 END) AS slow_units_deliv,
        SUM(max_capacity - units_on_hand) AS unused_capacity

    FROM amazon_potfolio.inventory iv
    JOIN amazon_potfolio.skus sk
        ON iv.sku_id = sk.sku_id
    JOIN amazon_potfolio.nodes nd
        ON iv.node_id = nd.node_id

    GROUP BY nd.node_type
)

SELECT
    node_type,
 
    fast_units,
    medium_units,
    slow_units,

    unused_capacity
 
    ROUND(fast_units  / (fast_units + medium_units + slow_units) * 100, 2) AS fast_percent,
    ROUND(medium_units  / (fast_units + medium_units + slow_units) * 100, 2) AS medium_percent,
    ROUND(slow_units  / (fast_units + medium_units + slow_units) * 100, 2) AS slow_percent

FROM base;

QUESTION 2.1
-- this is to check the rate of stockout from the FC, customer stockout is priority, stockout within the system/nodes is secondary 

SELECT 
    fulfillment_status,
    SUM(units_demanded) AS total_units,
    CAST(
        SUM(units_demanded) * 100.0 
        / SUM(SUM(units_demanded)) OVER ()
        AS DECIMAL(10,2)
    ) AS percentage
FROM amazon_potfolio.orders
GROUP BY fulfillment_status;

-- sku stockout rate based of velocity of sku
SELECT 
	 sk.velocity,
    od.fulfillment_status,
    SUM(od.units_demanded) AS total_units,
    CAST(
        SUM(od.units_demanded) * 100.0 
        / SUM(SUM(od.units_demanded)) OVER ()
        AS DECIMAL(10,2)
    ) AS percentage
FROM amazon_potfolio.orders od
JOIN amazon_potfolio.skus sk
	on od.sku_id = sk.sku_id
where fulfillment_status = 'stockout'
GROUP BY sk.velocity, od.fulfillment_status;

-- stockout rate based on node/ location 
SELECT fc_node,
	   count(fc_node)
FROM amazon_potfolio.orders
where fulfillment_status = 'stockout'
group by fc_node;


-- stockout rate, sku based on velocity and units demanded
SELECT 
	   od.sku_id,
       sk.velocity,
	   count(od.sku_id),
       sum(od.units_demanded)
FROM amazon_potfolio.orders od
JOIN amazon_potfolio.skus sk
	on od.sku_id = sk.sku_id
where fulfillment_status = 'stockout'
group by od.sku_id, sk.velocity;

-- stockout rate based on promo period 
SELECT is_promotional_period,
	   count(*),
       sum(units_demanded)
FROM amazon_potfolio.orders
where fulfillment_status = 'stockout'
group by is_promotional_period;


QUESTION 3.1

-- TO check if we have some sku that stockout upstream. 
SELECT 
	  'stockout',
	  SUM(case when po_id is not null then 1 else 0 end) as order_placed,
      ROUND(SUM(case when po_id is not null then 1 else 0 end)/ count(*) , 5) as order_percentage,
      SUM(CASE WHEN po_id is null then 1 else 0 end) as order_no_placed,
      ROUND(SUM(case when po_id is null then 1 else 0 end)/ count(*) , 5) as order_placed_percentage,
      count(*) as total_order
FROM amazon_potfolio.orders od
LEFT JOIN amazon_potfolio.purchase_orders pr
	ON od.sku_id = pr.sku_id
    AND od.order_date = pr.eta_date
WHERE fulfillment_status = 'stockout'


-- this established that these sku are in the system but they are just wrongly places.
SELECT
	  od.fc_node,
      od.units_demanded,
      od.sku_id, 
      tr.to_node,
      tr.from_node,
      tr.units,
      tr.status
FROM amazon_potfolio.orders OD
JOIN amazon_potfolio.transfers TR
	ON od.sku_id = tr.sku_id
    and od.order_date = tr.transfer_date
where od.fulfillment_status = 'stockout'

-- shows that sku are wrongly delivered to wrong node in case stockout
SELECT
	  od.fc_node,
      SUM(CASE WHEN od.fc_node = tr.to_node then 1 else 0 end) as correct_delivery, 
      SUM(CASE WHEN od.fc_node != tr.to_node then 1 else 0 end) as incorrect_delivery
FROM amazon_potfolio.orders OD
JOIN amazon_potfolio.transfers TR
	ON od.sku_id = tr.sku_id
    and od.order_date = tr.transfer_date
where od.fulfillment_status = 'stockout'
group by od.fc_node

-- this is kinda of repeating that all most of the stockouts are withing the system 
SELECT 
	fulfillment_status AS no_sale , 
        status as within_system,
	COUNT(*)
FROM amazon_potfolio.orders od
LEFT JOIN amazon_potfolio.purchase_orders pr
	ON od.sku_id = pr.sku_id
    AND od.order_date = pr.eta_date
group by fulfillment_status, status;

-- this the next step, this check the transfers in the system to look for stock out if there where available skus in the system 
SELECT 
	   fulfillment_status as customer_order,
	   status as system_order,
	   count(*)
FROM amazon_potfolio.orders OD
LEFT JOIN amazon_potfolio.transfers TR
	ON OD.sku_id = TR.sku_id
    AND OD.order_date = TR.transfer_date
WHERE fulfillment_status = 'stockout'
GROUP BY  fulfillment_status, status;

-- THIS IS SHOWING US THE COST FOR DIFFERENT MODES OF TRANSPORT 
SELECT
       transfer_mode,
       count(*),
	   cast(AVG(handling_cost_eur) as decimal(10,2)),
       cast(AVG(transit_cost_eur)as decimal(10,2)),
       cast(AVG(storage_cost_eur)as decimal(10,2)),
       cast(SUM(total_cost_eur) as decimal(10,2)),
       cast(AVG(total_cost_eur) as decimal(10,2))
FROM amazon_potfolio.transfers
GROUP BY transfer_mode

-- this is to see which route has the higher chance of failures 
SELECT 
    CONCAT(from_node, '-', to_node) AS route,
    SUM(CASE WHEN sla_breached = 'TRUE' THEN 1 ELSE 0 END) AS no_of_failures 
FROM amazon_potfolio.sla_performance
GROUP BY CONCAT(from_node, '-', to_node)
ORDER BY no_of_failures DESC;

-- this is to see which failures occur the most 
SELECT distinct delay_reason, 
	count(delay_reason) as reason
FROM amazon_potfolio.sla_performance
group by delay_reason
order by reason desc

-- THIS SHOW SHOWS THE MOST EFFIECIENT AND LEAST EFFIECIENT IXD
WITH base AS (
    SELECT 
        CASE 
            WHEN LOWER(from_node) LIKE 'ixd%' THEN from_node
            WHEN LOWER(to_node) LIKE 'ixd%' THEN to_node
            ELSE NULL
        END AS node,

        transfer_date,

        SUM(CASE WHEN LOWER(to_node) LIKE 'ixd%' THEN units ELSE 0 END) AS inbound,

        SUM(CASE WHEN LOWER(from_node) LIKE 'ixd%' THEN units ELSE 0 END) AS outbound,

        SUM(CASE WHEN LOWER(from_node) LIKE 'ixd%' THEN units ELSE 0 END) -
        SUM(CASE WHEN LOWER(to_node) LIKE 'ixd%' THEN units ELSE 0 END) AS balance,
        
        AVG(nd.capacity_units) AS node_capacity   -- 🔥 important

    FROM amazon_potfolio.transfers TR
    JOIN amazon_potfolio.nodes ND
        ON TR.to_node = ND.node_id

    GROUP BY 
        CASE 
            WHEN LOWER(from_node) LIKE 'ixd%' THEN from_node
            WHEN LOWER(to_node) LIKE 'ixd%' THEN to_node
            ELSE NULL
        END,
        transfer_date
),

running AS (
    SELECT 
        node,
        transfer_date,
        inbound,
        outbound,
        balance,
        node_capacity,

        SUM(balance) OVER (
            PARTITION BY node
            ORDER BY transfer_date
        ) AS cumulative_balance

    FROM base
    WHERE node IS NOT NULL
)

SELECT 
    node,
    MIN(node_capacity) AS node_capacity,   -- 🔥 safe aggregation
    MIN(cumulative_balance) AS min_cumulative_balance,
    MAX(cumulative_balance) AS max_cumulative_balance
FROM running
GROUP BY node
ORDER BY node;

-- this show downstream, how the pressure on the IXD affect the stockout rate for the final customer
SELECT TR.from_node,
		OD.fulfillment_status,
        COUNT(*),
        SUM(OD.units_demanded)
FROM amazon_potfolio.orders OD
LEFT JOIN  amazon_potfolio.transfers TR
ON OD.fc_node = TR.to_node
AND OD.sku_id = TR.sku_id
GROUP BY TR.from_node,
		OD.fulfillment_status;

QUESTION 4
-- THIS IS THE WHOLE OF SECITON 4 INTO ONE QUERY 
SELECT 
    rc.ixd_node,

    SUM(rc.push_qty_ordered) AS total_push, 
    SUM(rc.pull_qty_ordered) AS total_pull,

    AVG(rc.push_qty_ordered) AS avg_push,
    AVG(rc.pull_qty_ordered) AS avg_pull,

    ROUND(
        SUM(CASE WHEN rc.push_stockout = 'TRUE' THEN 1 ELSE 0 END) * 1.0 
        / COUNT(*) * 100,
    2) AS push_stockout_rate,

    ROUND(
        SUM(CASE WHEN rc.pull_stockout = 'TRUE' THEN 1 ELSE 0 END) * 1.0 
        / COUNT(*) * 100,
    2) AS pull_stockout_rate,

    -- SUM(rc.push_idle_days) AS total_push_idle,
    -- SUM(rc.pull_idle_days) AS total_pull_idle,

    CAST(AVG(rc.push_idle_days) AS DECIMAL(10,2)) AS avg_push_idle,
    CAST(AVG(rc.pull_idle_days) AS DECIMAL(10,2)) AS avg_pull_idle,
    
    -- Dynamic idle time  (min of push vs pull)
    CAST(SUM(
        CASE 
            WHEN rc.push_idle_days >= rc.pull_idle_days THEN rc.pull_idle_days
            ELSE rc.push_idle_days
        END
    ) AS DECIMAL(10,2)) AS total_min_dynamic_idle_days,
    
    CAST(AVG(
        CASE 
            WHEN rc.push_idle_days >= rc.pull_idle_days THEN rc.pull_idle_days
            ELSE rc.push_idle_days
        END
    ) AS DECIMAL(10,2)) AS avg_dynamic_idle_days,

    CAST(SUM(rc.pull_idle_days) -
    SUM(
        CASE 
            WHEN rc.push_idle_days >= rc.pull_idle_days THEN rc.pull_idle_days
            ELSE rc.push_idle_days
        END
    ) AS DECIMAL(10,2)) AS pull_dynamic_pull_idle_days_savings,
    
    CAST(SUM(rc.push_total_cost_eur) AS DECIMAL(10,2)) AS total_push_cost,
    CAST(SUM(rc.pull_total_cost_eur) AS DECIMAL(10,2)) AS total_pull_cost,

	CAST(AVG(rc.push_total_cost_eur) AS DECIMAL(10,2)) AS avg_push_cost,
    CAST(AVG(rc.pull_total_cost_eur) AS DECIMAL(10,2)) AS avg_pull_cost,

  -- Dynamic cost (min of push vs pull)
    CAST(SUM(
        CASE 
            WHEN rc.push_total_cost_eur >= rc.pull_total_cost_eur THEN rc.pull_total_cost_eur
            ELSE rc.push_total_cost_eur
        END
    ) AS DECIMAL(10,2)) AS total_dynamic_cost,

    CAST(AVG(
        CASE 
            WHEN rc.push_total_cost_eur >= rc.pull_total_cost_eur THEN rc.pull_total_cost_eur
            ELSE rc.push_total_cost_eur
        END
    ) AS DECIMAL(10,2)) AS avg_dynamic_cost,

    -- FIX: recompute instead of using alias
    CAST(SUM(rc.push_total_cost_eur) -
    SUM(
        CASE 
            WHEN rc.push_total_cost_eur >= rc.pull_total_cost_eur THEN rc.pull_total_cost_eur
            ELSE rc.push_total_cost_eur
        END
    ) AS DECIMAL(10,2)) AS total_dynamic_push_cost_savings,

    CAST(SUM(rc.pull_total_cost_eur) -
    SUM(
        CASE 
            WHEN rc.push_total_cost_eur >= rc.pull_total_cost_eur THEN rc.pull_total_cost_eur
            ELSE rc.push_total_cost_eur
        END
    ) AS DECIMAL(10,2)) AS total_dynamic_pull_cost_savings

FROM amazon_potfolio.replenishment_comparison rc
JOIN amazon_potfolio.skus sk
    ON rc.sku_id = sk.sku_id

GROUP BY rc.ixd_node;

-- THIS IS THE MAJOR COMPONENT OF THE DYNAMIC SELECTION STATMEMENT, Its a decision btw stockout and cost, and you need 2 senerios prioritize cost and prioritize service level, and the best situation in both cases. 

 