select 
    b.seq_number,
    b.city_code,
    b.valid,
    b.signed_state,
    b.user_id,
    b.num
from (
    select
        seq_number,
        valid,
        signed_state,
        user_id,
        row_number() over(
            partition by 
                seq_number 
            order by 
                seq_number,
                valid desc,
                instr(signed_state,'120'), 
                user_id desc
        ) as num
    from ods_crm_crm_customer 
    where dt='20160917' 
    and seq_number!=''
) b where b.num=1

select 
    id,
    connect_id,
    from_date,
    to_date,
    piece,
    update_time，
	row_number() over(
        partition by 
            id,
			optype
        order by 
            create_time desc
    ) as num
from ods_qta_price_item_change_log_orc
where dt='20161113'

