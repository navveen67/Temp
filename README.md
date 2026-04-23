-- 1. Drop the existing sequence if it exists (Optional/CAUTION)
-- DROP SEQUENCE IF EXISTS sdk_key_ID_SEQ;

-- 2. Create the sequence with INCREMENT BY 50
CREATE SEQUENCE sdk_key_ID_SEQ
    START WITH 1
    INCREMENT BY 50
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;
