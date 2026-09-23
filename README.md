# TecnoFix-PL-SQL

<details>
<summary>▶ Haz clic para ver el código de PL/SQL</summary>

```sql
SET SERVEROUTPUT ON;

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
               v_cliente_nombre := 'DATO NO REGISTRADO';
            WHEN TOO_MANY_ROWS THEN
               v_cliente_nombre := 'ERROR: RUT DUPLICADO EN CLIENTE';
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

    CREATE OR REPLACE PROCEDURE actualizar_stock_repuesto (
      p_repuesto_id  IN repuesto.repuesto_id%TYPE,
      p_cantidad     IN NUMBER
    ) IS
    BEGIN
       UPDATE repuesto
       SET stock_actual = stock_actual - p_cantidad
       WHERE repuesto_id = p_repuesto_id;
    END actualizar_stock_repuesto;
    /

    CREATE OR REPLACE FUNCTION valor_total_reparacion (
      p_reparacion_id IN reparacion.reparacion_id%TYPE
    ) RETURN NUMBER IS
       v_total NUMBER := 0;
    BEGIN
       SELECT NVL(SUM(rr.cantidad_utilizada * rp.precio_unitario), 0)
       INTO v_total
       FROM reparacion_repuesto rr
       JOIN repuesto rp ON rp.repuesto_id = rr.repuesto_id
       WHERE rr.reparacion_id = p_reparacion_id;
     
       RETURN v_total;
    END valor_total_reparacion;
    /

    CREATE OR REPLACE PACKAGE pkg_tecnofix IS
       PROCEDURE actualizar_stock_repuesto(p_repuesto_id NUMBER, p_cantidad NUMBER);
       FUNCTION valor_total_reparacion(p_reparacion_id NUMBER) RETURN NUMBER;
    END pkg_tecnofix;
    /



    CREATE OR REPLACE TRIGGER trg_auditoria_tecnico
    AFTER UPDATE OF habilitado ON tecnico
    FOR EACH ROW
    BEGIN
       INSERT INTO auditoria_cambios (
         tabla_afectada, registro_id, campo_modificado,
         valor_anterior, valor_nuevo, operacion,
         fecha_modificacion, usuario_responsable
       ) VALUES (
         'TECNICO', :OLD.tecnico_id, 'habilitado',
         :OLD.habilitado, :NEW.habilitado, 'UPDATE',
         SYSTIMESTAMP, USER
       );
    END trg_auditoria_tecnico;
    /




