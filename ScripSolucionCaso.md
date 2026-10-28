# TecnoFix-PL-SQL

<details>
<summary>▶ Haz clic para ver el código de PL/SQL</summary>

```sql


SET SERVEROUTPUT ON;


-- 1. SOLUCION BASE (EVALUACION 1)

DECLARE

   TYPE r_repuesto_rec IS RECORD (
      repuesto_id       repuesto.repuesto_id%TYPE,
      nombre_repuesto   repuesto.nombre_repuesto%TYPE,
      cantidad          reparacion_repuesto.cantidad_utilizada%TYPE,
      precio_unitario   repuesto.precio_unitario%TYPE,
      subtotal          NUMBER
   );
   v_repuesto   r_repuesto_rec;

   TYPE varray_descuentos IS VARRAY(2) OF NUMBER;
   v_descuentos   varray_descuentos := varray_descuentos(0, 0);

   CURSOR c_tecnicos IS
      SELECT tecnico_id, nombre, apellido
      FROM tecnico
      WHERE habilitado = 'S';

   CURSOR c_ordenes(p_tecnico_id NUMBER) IS
      SELECT o.orden_id,
             o.cliente_id,
             cl.fecha_registro,
             r.reparacion_id
      FROM orden_servicio o
      JOIN reparacion r      ON r.orden_id = o.orden_id
      JOIN cliente cl        ON cl.cliente_id = o.cliente_id
      WHERE o.tecnico_id = p_tecnico_id
        AND r.resultado_reparacion = 'EXITOSA';

   CURSOR c_repuestos(p_reparacion_id NUMBER) IS
      SELECT rp.repuesto_id,
             rp.nombre_repuesto,
             rr.cantidad_utilizada,
             rp.precio_unitario
      FROM reparacion_repuesto rr
      JOIN repuesto rp ON rp.repuesto_id = rr.repuesto_id
      WHERE rr.reparacion_id = p_reparacion_id;


   v_cliente_nombre        cliente.nombre%TYPE;
   v_valor_total_repuestos NUMBER;
   v_cantidad_total        NUMBER;
   v_promedio_unitario     NUMBER;
   v_antiguedad_anios      NUMBER;
   v_total_descuento       NUMBER;
   v_valor_final           NUMBER;
   v_procesar_orden        BOOLEAN;
   v_ordenes_procesadas    NUMBER := 0;

   e_descuento_excede_limite EXCEPTION;
   e_orden_sin_repuestos     EXCEPTION;

BEGIN
   EXECUTE IMMEDIATE 'TRUNCATE TABLE resumen_reparacion_tecnico';

   FOR t IN c_tecnicos LOOP

      FOR o IN c_ordenes(t.tecnico_id) LOOP

         v_valor_total_repuestos := 0;
         v_cantidad_total        := 0;
         v_procesar_orden        := TRUE;


         BEGIN
            SELECT nombre || ' ' || apellido
            INTO v_cliente_nombre
            FROM cliente
            WHERE cliente_id = o.cliente_id;
         EXCEPTION
            WHEN NO_DATA_FOUND THEN
               v_cliente_nombre := 'CLIENTE NO REGISTRADO';
            WHEN TOO_MANY_ROWS THEN
               v_cliente_nombre := 'ERROR: DATO DUPLICADO EN CLIENTE';
         END;


         FOR rep IN c_repuestos(o.reparacion_id) LOOP
            v_repuesto.repuesto_id     := rep.repuesto_id;
            v_repuesto.nombre_repuesto := rep.nombre_repuesto;
            v_repuesto.cantidad        := rep.cantidad_utilizada;
            v_repuesto.precio_unitario := rep.precio_unitario;
            v_repuesto.subtotal        := rep.cantidad_utilizada * rep.precio_unitario;

            v_valor_total_repuestos := v_valor_total_repuestos + v_repuesto.subtotal;
            v_cantidad_total        := v_cantidad_total + v_repuesto.cantidad;
         END LOOP;


         BEGIN
            IF v_cantidad_total = 0 THEN
               RAISE e_orden_sin_repuestos;
            END IF;
         EXCEPTION
            WHEN e_orden_sin_repuestos THEN
               v_procesar_orden := FALSE;
               DBMS_OUTPUT.PUT_LINE('Orden ' || o.orden_id ||
                  ' omitida: reparacion exitosa sin repuestos registrados.');
         END;

         IF v_procesar_orden THEN

            BEGIN
               v_promedio_unitario := v_valor_total_repuestos / v_cantidad_total;
            EXCEPTION
               WHEN ZERO_DIVIDE THEN
                  v_promedio_unitario := 0;
            END;

            IF v_cantidad_total >= 5 THEN
               v_descuentos(1) := ROUND(v_valor_total_repuestos * 0.10);
            ELSIF v_cantidad_total >= 2 THEN
               v_descuentos(1) := ROUND(v_valor_total_repuestos * 0.05);
            ELSE
               v_descuentos(1) := 0;
            END IF;

            v_antiguedad_anios := TRUNC(MONTHS_BETWEEN(SYSDATE, o.fecha_registro) / 12);
            IF v_antiguedad_anios >= 2 THEN
               v_descuentos(2) := ROUND(v_valor_total_repuestos * 0.05);
            ELSE
               v_descuentos(2) := 0;
            END IF;

            v_total_descuento := v_descuentos(1) + v_descuentos(2);

            BEGIN
               IF v_total_descuento > ROUND(v_valor_total_repuestos * 0.20) THEN
                  RAISE e_descuento_excede_limite;
               END IF;
            EXCEPTION
               WHEN e_descuento_excede_limite THEN
                  v_total_descuento := ROUND(v_valor_total_repuestos * 0.20);
                  DBMS_OUTPUT.PUT_LINE('Orden ' || o.orden_id ||
                     ': descuento topeado al 20% del valor de repuestos.');
            END;

            v_valor_final := v_valor_total_repuestos - v_total_descuento;

            INSERT INTO resumen_reparacion_tecnico (
               tecnico_id, tecnico_nombre, orden_id, reparacion_id,
               cliente_id, cliente_nombre, cantidad_repuestos, valor_repuestos,
               descuento_volumen, descuento_antiguedad, descuento_total, valor_final
            ) VALUES (
               t.tecnico_id, t.nombre || ' ' || t.apellido, o.orden_id, o.reparacion_id,
               o.cliente_id, v_cliente_nombre, v_cantidad_total, v_valor_total_repuestos,
               v_descuentos(1), v_descuentos(2), v_total_descuento, v_valor_final
            );

            v_ordenes_procesadas := v_ordenes_procesadas + 1;
         END IF;

      END LOOP; 
   END LOOP; 

   COMMIT;
   DBMS_OUTPUT.PUT_LINE('Proceso finalizado. Ordenes procesadas: ' || v_ordenes_procesadas);

EXCEPTION

   WHEN OTHERS THEN
      ROLLBACK;
      DBMS_OUTPUT.PUT_LINE('Error inesperado: ' || SQLCODE || ' - ' || SQLERRM);
      RAISE;
END;
/

SELECT * FROM resumen_reparacion_tecnico ORDER BY tecnico_id, orden_id;


-- 2. FUNCIONES

CREATE OR REPLACE FUNCTION fn_total_valor_final
RETURN NUMBER
IS
   v_total NUMBER;
BEGIN
   SELECT NVL(SUM(valor_final), 0) INTO v_total
   FROM resumen_reparacion_tecnico;
   RETURN v_total;
END;
/

CREATE OR REPLACE FUNCTION fn_valor_repuestos_reparacion (p_reparacion_id IN NUMBER)
RETURN NUMBER
IS
   v_valor NUMBER;
BEGIN
   SELECT NVL(SUM(rr.cantidad_utilizada * rp.precio_unitario), 0) INTO v_valor
   FROM reparacion_repuesto rr
   JOIN repuesto rp ON rp.repuesto_id = rr.repuesto_id
   WHERE rr.reparacion_id = p_reparacion_id;
   RETURN v_valor;
END;
/

CREATE OR REPLACE FUNCTION fn_cantidad_repuestos_reparacion (p_reparacion_id IN NUMBER)
RETURN NUMBER
IS
   v_cantidad NUMBER;
BEGIN
   SELECT NVL(SUM(cantidad_utilizada), 0) INTO v_cantidad
   FROM reparacion_repuesto
   WHERE reparacion_id = p_reparacion_id;
   RETURN v_cantidad;
END;
/

CREATE OR REPLACE FUNCTION fn_descuento_volumen (p_cantidad IN NUMBER, p_valor IN NUMBER)
RETURN NUMBER
IS
BEGIN
   IF p_cantidad >= 5 THEN
      RETURN ROUND(p_valor * 0.10);
   ELSIF p_cantidad >= 2 THEN
      RETURN ROUND(p_valor * 0.05);
   END IF;
   RETURN 0;
END;
/

CREATE OR REPLACE FUNCTION fn_descuento_antiguedad (p_fecha_registro IN DATE, p_valor IN NUMBER)
RETURN NUMBER
IS
BEGIN
   IF MONTHS_BETWEEN(SYSDATE, p_fecha_registro) >= 24 THEN
      RETURN ROUND(p_valor * 0.05);
   END IF;
   RETURN 0;
END;
/


-- 3. PROCEDIMIENTOS

CREATE OR REPLACE PROCEDURE sp_resumen_tecnico (
   p_tecnico_id IN  NUMBER,
   p_ordenes    OUT NUMBER
) AS
   CURSOR c_ordenes IS
      SELECT o.orden_id, o.cliente_id, r.reparacion_id,
             cl.nombre || ' ' || cl.apellido AS cliente_nombre, cl.fecha_registro
      FROM orden_servicio o
      JOIN reparacion r ON r.orden_id = o.orden_id
      JOIN cliente cl   ON cl.cliente_id = o.cliente_id
      WHERE o.tecnico_id = p_tecnico_id
        AND r.resultado_reparacion = 'EXITOSA';

   v_tecnico_nombre  VARCHAR2(120);
   v_cantidad        NUMBER;
   v_valor           NUMBER;
   v_desc_volumen    NUMBER;
   v_desc_antiguedad NUMBER;
   v_desc_total      NUMBER;
BEGIN
   p_ordenes := 0;

   SELECT nombre || ' ' || apellido INTO v_tecnico_nombre
   FROM tecnico
   WHERE tecnico_id = p_tecnico_id;

   DELETE FROM resumen_reparacion_tecnico WHERE tecnico_id = p_tecnico_id;

   FOR o IN c_ordenes LOOP
      v_cantidad := fn_cantidad_repuestos_reparacion(o.reparacion_id);
      v_valor    := fn_valor_repuestos_reparacion(o.reparacion_id);

      IF v_cantidad > 0 THEN
         v_desc_volumen    := fn_descuento_volumen(v_cantidad, v_valor);
         v_desc_antiguedad := fn_descuento_antiguedad(o.fecha_registro, v_valor);
         v_desc_total      := LEAST(v_desc_volumen + v_desc_antiguedad, ROUND(v_valor * 0.20));

         INSERT INTO resumen_reparacion_tecnico (
            tecnico_id, tecnico_nombre, orden_id, reparacion_id, cliente_id, cliente_nombre,
            cantidad_repuestos, valor_repuestos, descuento_volumen, descuento_antiguedad,
            descuento_total, valor_final
         ) VALUES (
            p_tecnico_id, v_tecnico_nombre, o.orden_id, o.reparacion_id, o.cliente_id, o.cliente_nombre,
            v_cantidad, v_valor, v_desc_volumen, v_desc_antiguedad,
            v_desc_total, v_valor - v_desc_total
         );

         p_ordenes := p_ordenes + 1;
      ELSE
         DBMS_OUTPUT.PUT_LINE('Orden ' || o.orden_id || ' omitida: sin repuestos registrados.');
      END IF;
   END LOOP;
EXCEPTION
   WHEN NO_DATA_FOUND THEN
      RAISE_APPLICATION_ERROR(-20010, 'El tecnico ' || p_tecnico_id || ' no existe');
END;
/

CREATE OR REPLACE PROCEDURE sp_generar_resumen_reparaciones AS
   v_ordenes NUMBER;
   v_total   NUMBER := 0;
BEGIN
   DELETE FROM resumen_reparacion_tecnico;

   FOR t IN (SELECT tecnico_id FROM tecnico WHERE habilitado = 'S') LOOP
      sp_resumen_tecnico(t.tecnico_id, v_ordenes);
      v_total := v_total + v_ordenes;
   END LOOP;

   COMMIT;
   DBMS_OUTPUT.PUT_LINE('Resumen generado. Ordenes procesadas: ' || v_total);
END;
/

CREATE OR REPLACE PROCEDURE sp_actualizar_stock_repuesto (
   p_repuesto_id IN NUMBER,
   p_cantidad    IN NUMBER
) AS
   v_stock repuesto.stock_actual%TYPE;
BEGIN
   SELECT stock_actual INTO v_stock
   FROM repuesto
   WHERE repuesto_id = p_repuesto_id;

   IF v_stock < p_cantidad THEN
      RAISE_APPLICATION_ERROR(-20022, 'Stock insuficiente: disponible ' || v_stock);
   END IF;

   UPDATE repuesto
   SET stock_actual = stock_actual - p_cantidad
   WHERE repuesto_id = p_repuesto_id;
END;
/

-- 4. PACKAGE pkg_tecnofix

CREATE OR REPLACE PACKAGE pkg_tecnofix AS
   g_usuario VARCHAR2(60);                      

   PROCEDURE ejecutar_proceso;
   FUNCTION valor_final_reparacion (p_reparacion_id IN NUMBER) RETURN NUMBER;
   FUNCTION total_tecnico (p_tecnico_id IN NUMBER) RETURN NUMBER;
END pkg_tecnofix;
/

CREATE OR REPLACE PACKAGE BODY pkg_tecnofix AS
   g_ejecuciones NUMBER;                        

   PROCEDURE p_log (p_mensaje IN VARCHAR2) IS
   BEGIN
      DBMS_OUTPUT.PUT_LINE('[' || TO_CHAR(SYSDATE, 'HH24:MI:SS') || '] ' || p_mensaje);
   END;

   PROCEDURE ejecutar_proceso IS
   BEGIN
      g_ejecuciones := g_ejecuciones + 1;
      p_log('Ejecucion ' || g_ejecuciones || ' (usuario ' || g_usuario || ')');
      sp_generar_resumen_reparaciones;
      p_log('Valor final total: ' || fn_total_valor_final);
   END;

   FUNCTION valor_final_reparacion (p_reparacion_id IN NUMBER) RETURN NUMBER IS
      v_valor    NUMBER := fn_valor_repuestos_reparacion(p_reparacion_id);
      v_cantidad NUMBER := fn_cantidad_repuestos_reparacion(p_reparacion_id);
      v_fecha    DATE;
   BEGIN
      SELECT c.fecha_registro INTO v_fecha
      FROM reparacion r
      JOIN orden_servicio o ON o.orden_id = r.orden_id
      JOIN cliente c        ON c.cliente_id = o.cliente_id
      WHERE r.reparacion_id = p_reparacion_id;

      RETURN v_valor - LEAST(fn_descuento_volumen(v_cantidad, v_valor)
                           + fn_descuento_antiguedad(v_fecha, v_valor), ROUND(v_valor * 0.20));
   END;

   FUNCTION total_tecnico (p_tecnico_id IN NUMBER) RETURN NUMBER IS
      v_total NUMBER;
   BEGIN
      SELECT NVL(SUM(valor_final), 0) INTO v_total
      FROM resumen_reparacion_tecnico
      WHERE tecnico_id = p_tecnico_id;
      RETURN v_total;
   END;


BEGIN
   g_usuario     := USER;
   g_ejecuciones := 0;
END pkg_tecnofix;
/

-- 5. TRIGGERS

CREATE OR REPLACE TRIGGER trg_orden_tecnico_habilitado
BEFORE INSERT OR UPDATE OF tecnico_id ON orden_servicio
FOR EACH ROW
WHEN (NEW.tecnico_id IS NOT NULL)
DECLARE
   v_habilitado tecnico.habilitado%TYPE;
BEGIN
   SELECT habilitado INTO v_habilitado
   FROM tecnico
   WHERE tecnico_id = :NEW.tecnico_id;

   IF v_habilitado <> 'S' THEN
      RAISE_APPLICATION_ERROR(-20001, 'El tecnico no esta habilitado');
   END IF;
END;
/

CREATE OR REPLACE TRIGGER trg_orden_control_estado
BEFORE UPDATE OF estado_id ON orden_servicio
FOR EACH ROW
DECLARE
   v_final estado_orden.es_estado_final%TYPE;
BEGIN
   SELECT es_estado_final INTO v_final
   FROM estado_orden
   WHERE estado_id = :OLD.estado_id;

   IF v_final = 'S' THEN
      RAISE_APPLICATION_ERROR(-20003, 'La orden esta en estado final');
   END IF;
END;
/

CREATE OR REPLACE TRIGGER trg_orden_auditoria
AFTER UPDATE OF estado_id ON orden_servicio
FOR EACH ROW
BEGIN
   INSERT INTO historial_estado_orden (orden_id, estado_anterior_id, estado_nuevo_id,
                                       usuario_responsable, observacion)
   VALUES (:NEW.orden_id, :OLD.estado_id, :NEW.estado_id, USER, 'Cambio registrado por trigger');
END;
/

CREATE OR REPLACE TRIGGER trg_reparacion_repuesto_stock
BEFORE INSERT ON reparacion_repuesto
FOR EACH ROW
BEGIN
   sp_actualizar_stock_repuesto(:NEW.repuesto_id, :NEW.cantidad_utilizada);
END;
/

CREATE OR REPLACE TRIGGER trg_resumen_valida
BEFORE INSERT OR UPDATE ON resumen_reparacion_tecnico
FOR EACH ROW
BEGIN
   IF :NEW.descuento_total > ROUND(:NEW.valor_repuestos * 0.20) THEN
      RAISE_APPLICATION_ERROR(-20007, 'El descuento supera el 20% del valor de repuestos');
   END IF;
END;
/

CREATE OR REPLACE TRIGGER trg_orden_no_eliminar
BEFORE DELETE ON orden_servicio
BEGIN
   RAISE_APPLICATION_ERROR(-20005, 'No se permite eliminar ordenes de servicio');
END;
/

SHOW ERRORS;


-- 6. PRUEBAS 

-- Funciones 
SELECT fn_total_valor_final AS valor_final_total FROM dual;

SELECT r.reparacion_id,
       fn_cantidad_repuestos_reparacion(r.reparacion_id)    AS cant_repuestos,
       fn_valor_repuestos_reparacion(r.reparacion_id)       AS valor_repuestos,
       pkg_tecnofix.valor_final_reparacion(r.reparacion_id) AS valor_final
FROM reparacion r
WHERE r.resultado_reparacion = 'EXITOSA'
ORDER BY r.reparacion_id
FETCH FIRST 10 ROWS ONLY;

-- Procedimientos
EXEC sp_generar_resumen_reparaciones;
SELECT * FROM resumen_reparacion_tecnico ORDER BY tecnico_id, orden_id;

DECLARE
   v_ordenes NUMBER;
BEGIN
   sp_resumen_tecnico(2, v_ordenes);
   DBMS_OUTPUT.PUT_LINE('Ordenes recalculadas del tecnico 2: ' || v_ordenes);
END;
/

-- Package
EXEC pkg_tecnofix.ejecutar_proceso;
SELECT tecnico_id, pkg_tecnofix.total_tecnico(tecnico_id) AS total_valor_final
FROM tecnico
ORDER BY tecnico_id;

-- Triggers: operaciones que deben ser rechazadas
BEGIN
   UPDATE orden_servicio SET tecnico_id = 4 WHERE orden_id = 7;
EXCEPTION WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Bloqueado: ' || SQLERRM);
END;
/

BEGIN
   UPDATE orden_servicio SET estado_id = 3 WHERE orden_id = 1;
EXCEPTION WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Bloqueado: ' || SQLERRM);
END;
/

BEGIN
   DELETE FROM orden_servicio WHERE orden_id = 1;
EXCEPTION WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Bloqueado: ' || SQLERRM);
END;
/

BEGIN
   INSERT INTO reparacion_repuesto (reparacion_id, repuesto_id, cantidad_utilizada)
   VALUES (2, 8, 99);
EXCEPTION WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Bloqueado: ' || SQLERRM);
END;
/

BEGIN
   INSERT INTO resumen_reparacion_tecnico (tecnico_id, tecnico_nombre, orden_id, reparacion_id,
      cliente_id, valor_repuestos, descuento_total, valor_final)
   VALUES (1, 'PRUEBA', 1, 1, 1, 100000, 50000, 50000);
EXCEPTION WHEN OTHERS THEN DBMS_OUTPUT.PUT_LINE('Bloqueado: ' || SQLERRM);
END;
/

-- Triggers: operaciones validas que dejan historial
UPDATE orden_servicio SET estado_id = 6 WHERE orden_id = 7;
INSERT INTO reparacion_repuesto (reparacion_id, repuesto_id, cantidad_utilizada) VALUES (2, 8, 3);

SELECT stock_actual FROM repuesto WHERE repuesto_id = 8;
SELECT * FROM historial_estado_orden WHERE orden_id = 7 ORDER BY historial_id;

ROLLBACK;



