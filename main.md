import flet as ft
import os
import qrcode
import io
import base64
from database import (
    obtener_productos, registrar_venta, agregar_producto_nuevo, 
    obtener_categorias, eliminar_producto, actualizar_producto,
    obtener_ventas_por_dia, obtener_estadisticas_generales
)
from printer import TicketPrinter, preparar_datos_venta
from datetime import datetime

def main(page: ft.Page):
    page.title = "Inventario"
    page.padding = 0
    page.window.min_width = 1200
    page.bgcolor = ft.Colors.INDIGO_100

    printer = TicketPrinter(
        empresa="INVENTARIO LOCAL",
        direccion="Av. Principal 123",
        telefono="+51 963 499 142"
    )

    productos = obtener_productos()
    carrito = []
    total_text = ft.Text("S/0", size=30, weight="bold", color=ft.Colors.ORANGE)
    items_count_text = ft.Text("0 items", color=ft.Colors.GREY_400)
    list_view_pedido = ft.ListView(expand=True)

    def actualizar_total():
        total = sum(p["Precio"] * p["Cantidad"] for p in carrito)
        cantidad = sum(p["Cantidad"] for p in carrito)
        total_text.value = f"S/{total:.2f}"
        items_count_text.value = f"{cantidad} items · {len(carrito)} productos"
        page.update()

    def agregar_producto_carrito(product):
        for p in carrito:
            if p["Name"] == product["Name"]:
                if p["Cantidad"] < p["Stock"]:
                    p["Cantidad"] += 1
                    render_pedido()
                else:
                    page.snack_bar = ft.SnackBar(
                        ft.Text(f"Stock insuficiente. Solo hay {p['Stock']} unidades"),
                        bgcolor=ft.Colors.RED_400
                    )
                    page.snack_bar.open = True
                    page.update()
                return
        
        if product["Stock"] > 0:
            carrito.append({**product, "Cantidad": 1})
            render_pedido()
        else:
            page.snack_bar = ft.SnackBar(
                ft.Text("Producto sin stock disponible"),
                bgcolor=ft.Colors.RED_400
            )
            page.snack_bar.open = True
            page.update()

    def restar_producto(p):
        if p["Cantidad"] > 1:
            p["Cantidad"] -= 1
        else:
            carrito.remove(p)
        render_pedido()

    def render_pedido():
        list_view_pedido.controls.clear()
        for p in carrito:
            list_view_pedido.controls.append(
                ft.Container(
                    padding=5,
                    content=ft.Row([
                        ft.Image(
                            p["Image"], 
                            width=50, 
                            height=50, 
                            border_radius=8,
                            fit=ft.ImageFit.COVER,
                            error_content=ft.Icon(ft.Icons.BROKEN_IMAGE, color=ft.Colors.RED)
                        ),
                        ft.Column([
                            ft.Text(p["Name"], weight="bold", size=13),
                            ft.Text(f"Precio: S/{p['Precio']}", color=ft.Colors.GREY_500, size=11)
                        ], expand=True),
                        ft.Row([
                            ft.IconButton(icon=ft.Icons.REMOVE, on_click=lambda e, pr=p: restar_producto(pr)),
                            ft.Text(str(p["Cantidad"])),
                            ft.IconButton(icon=ft.Icons.ADD, on_click=lambda e, pr=p: agregar_producto_carrito(pr))
                        ])
                    ])
                )
            )
        actualizar_total()

    grid_productos = ft.GridView(
        expand=True, runs_count=5, max_extent=220, child_aspect_ratio=0.8,
        spacing=10, run_spacing=10
    )

    def render_productos(categoria="Todo", busqueda=""):
        grid_productos.controls.clear()
        
        for product in productos:
            if categoria != "Todo" and product["Categoria"].lower() != categoria.lower():
                continue
            
            if busqueda and busqueda.lower() not in product["Name"].lower():
                continue
            
            grid_productos.controls.append(
                ft.Container(
                    bgcolor=ft.Colors.WHITE,
                    border_radius=10,
                    padding=10,
                    content=ft.Column([
                        ft.Container(
                            content=ft.Image(
                                product["Image"], 
                                width=150, 
                                height=100, 
                                border_radius=8,
                                fit=ft.ImageFit.COVER,
                                error_content=ft.Icon(ft.Icons.BROKEN_IMAGE, color=ft.Colors.RED)
                            ),
                        ),
                        ft.Text(product["Name"], weight="bold", size=14, max_lines=2),
                        ft.Row([
                            ft.Text(f"S/{product['Precio']}", color=ft.Colors.ORANGE_700, weight="bold"),
                            ft.Container(
                                bgcolor=ft.Colors.GREEN_50 if product['Stock'] > 5 else (ft.Colors.RED_50 if product['Stock'] > 0 else ft.Colors.GREY_200),
                                border_radius=5,
                                padding=ft.padding.symmetric(horizontal=6, vertical=2),
                                content=ft.Text(
                                    f"Stock: {product['Stock']}", 
                                    size=10, 
                                    color=ft.Colors.GREEN_700 if product['Stock'] > 5 else (ft.Colors.RED_700 if product['Stock'] > 0 else ft.Colors.GREY_600)
                                )
                            )
                        ], alignment=ft.MainAxisAlignment.SPACE_BETWEEN),
                        ft.ElevatedButton(
                            "Agregar", 
                            bgcolor=ft.Colors.ORANGE if product['Stock'] > 0 else ft.Colors.GREY_400,
                            color=ft.Colors.WHITE,
                            disabled=product['Stock'] == 0,
                            on_click=lambda e, pr=product: agregar_producto_carrito(pr)
                        )
                    ],
                        alignment=ft.MainAxisAlignment.CENTER,
                        horizontal_alignment=ft.CrossAxisAlignment.CENTER
                    )
                )
            )
        page.update()

    search_field = ft.TextField(
        hint_text="Buscar productos...",
        prefix_icon=ft.Icons.SEARCH,
        border_radius=10,
        filled=True,
        bgcolor=ft.Colors.WHITE,
        height=50,
        on_change=lambda e: render_productos("Todo", e.control.value)
    )

    filtros = ft.Container(
        bgcolor=ft.Colors.WHITE,
        height=55,
        border_radius=10,
        padding=10,
        content=ft.Row([
            ft.FilledButton("Todo", bgcolor=ft.Colors.ORANGE_100, color=ft.Colors.ORANGE_600, on_click=lambda e: render_productos("Todo")),
            ft.TextButton("Laptops", on_click=lambda e: render_productos("Laptops")),
            ft.TextButton("Componentes", on_click=lambda e: render_productos("Componentes")),
            ft.TextButton("Periféricos", on_click=lambda e: render_productos("Periféricos")),
            ft.TextButton("PC", on_click=lambda e: render_productos("PC")),
            ft.TextButton("Monitores", on_click=lambda e: render_productos("Monitores")),
            ft.TextButton("Impresoras", on_click=lambda e: render_productos("Impresoras")),
        ],
            alignment=ft.MainAxisAlignment.SPACE_AROUND
        )
    )

    contenido_central = ft.Column([search_field, filtros, grid_productos], expand=True, spacing=10)

    def volver_inicio(e=None):
        contenido_central.controls.clear()
        contenido_central.controls.extend([search_field, filtros, grid_productos])
        render_productos("Todo")
        page.update()

    def generar_qr(texto):
        """Genera un código QR y lo devuelve como base64"""
        qr = qrcode.QRCode(version=1, box_size=10, border=2)
        qr.add_data(texto)
        qr.make(fit=True)
        
        img = qr.make_image(fill_color="black", back_color="white")
        
        buffer = io.BytesIO()
        img.save(buffer, format='PNG')
        buffer.seek(0)
        
        return base64.b64encode(buffer.getvalue()).decode()

    def procesar_pago(e):
        if not carrito:
            page.snack_bar = ft.SnackBar(
                ft.Text("El carrito está vacío"),
                bgcolor=ft.Colors.RED_400
            )
            page.snack_bar.open = True
            page.update()
            return
        
        metodo_seleccionado = {"valor": "Efectivo"}
        
        def cerrar_dialogo_pago(e):
            dialogo_metodo.open = False
            page.update()
        
        def mostrar_qr_confirmacion(e):
            dialogo_metodo.open = False
            page.update()
            
            # Generar datos para QR
            total = sum(item['Precio'] * item['Cantidad'] for item in carrito)
            numero_venta_temp = f"V-{datetime.now().strftime('%Y%m%d%H%M%S')}"
            
            datos_qr = f"Venta: {numero_venta_temp}\nTotal: S/{total:.2f}\nMétodo: {metodo_seleccionado['valor']}"
            qr_base64 = generar_qr(datos_qr)
            
            def confirmar_venta_final(e):
                # AQUÍ SE REGISTRA LA VENTA Y SE DESCUENTA EL STOCK
                resultado = registrar_venta(carrito, metodo_pago=metodo_seleccionado["valor"])
                
                if resultado['success']:
                    dialogo_qr.open = False
                    page.update()
                    
                    # Generar ticket
                    items_venta = [{
                        'nombre': item['Name'],
                        'cantidad': item['Cantidad'],
                        'precio': item['Precio'],
                        'subtotal': item['Precio'] * item['Cantidad']
                    } for item in carrito]
                    
                    venta_data = preparar_datos_venta(
                        numero_venta=resultado['numero_venta'],
                        items=items_venta,
                        cliente_nombre="Cliente General",
                        subtotal=resultado['total'],
                        descuento=0,
                        total=resultado['total']
                    )
                    
                    try:
                        rutas = printer.imprimir_ticket(venta_data, formato='txt')
                    except:
                        rutas = {}
                    
                    def cerrar_confirmacion(e):
                        dialogo_confirmacion.open = False
                        page.update()
                        carrito.clear()
                        render_pedido()
                        nonlocal productos
                        productos = obtener_productos()
                        render_productos()
                    
                    dialogo_confirmacion = ft.AlertDialog(
                        title=ft.Text("✓ Venta Confirmada", color=ft.Colors.GREEN),
                        content=ft.Column([
                            ft.Text(f"Nº: {resultado['numero_venta']}", weight="bold"),
                            ft.Text(f"Total: S/{resultado['total']:.2f}", size=24, color=ft.Colors.ORANGE, weight="bold"),
                            ft.Text(f"Método: {metodo_seleccionado['valor']}", color=ft.Colors.GREY_700),
                            ft.Divider(),
                            ft.Text("✓ Stock actualizado en SQL Server", color=ft.Colors.GREEN_700, weight="bold"),
                        ], tight=True, height=200),
                        actions=[
                            ft.FilledButton("Cerrar", bgcolor=ft.Colors.GREEN, on_click=cerrar_confirmacion)
                        ]
                    )
                    
                    page.dialog = dialogo_confirmacion
                    dialogo_confirmacion.open = True
                    page.update()
                else:
                    page.snack_bar = ft.SnackBar(
                        ft.Text(f"Error: {resultado.get('error', 'Error desconocido')}"),
                        bgcolor=ft.Colors.RED_400
                    )
                    page.snack_bar.open = True
                    page.update()
            
            def cancelar_venta(e):
                dialogo_qr.open = False
                page.update()
            
            dialogo_qr = ft.AlertDialog(
                title=ft.Text("📱 Escanea el código QR", text_align=ft.TextAlign.CENTER),
                content=ft.Column([
                    ft.Image(
                        src_base64=qr_base64,
                        width=250,
                        height=250,
                        fit=ft.ImageFit.CONTAIN
                    ),
                    ft.Divider(),
                    ft.Text(f"Total a pagar: S/{total:.2f}", size=18, weight="bold", text_align=ft.TextAlign.CENTER),
                    ft.Text(f"Método: {metodo_seleccionado['valor']}", size=14, color=ft.Colors.GREY_600, text_align=ft.TextAlign.CENTER),
                ], tight=True, height=380, horizontal_alignment=ft.CrossAxisAlignment.CENTER),
                actions=[
                    ft.TextButton("Cancelar", on_click=cancelar_venta),
                    ft.FilledButton("✓ Confirmar Pago", bgcolor=ft.Colors.GREEN, on_click=confirmar_venta_final)
                ],
                actions_alignment=ft.MainAxisAlignment.SPACE_BETWEEN
            )
            
            page.dialog = dialogo_qr
            dialogo_qr.open = True
            page.update()
        
        dialogo_metodo = ft.AlertDialog(
            title=ft.Text("💳 Método de Pago", size=18, weight="bold"),
            content=ft.Column([
                ft.RadioGroup(
                    content=ft.Column([
                        ft.Radio(value="Efectivo", label="💵 Efectivo"),
                        ft.Radio(value="Tarjeta", label="💳 Tarjeta"),
                        ft.Radio(value="Yape", label="📱 Yape/Plin"),
                        ft.Radio(value="Transferencia", label="🏦 Transferencia"),
                    ]),
                    value="Efectivo",
                    on_change=lambda e: metodo_seleccionado.update({"valor": e.control.value})
                )
            ], tight=True, height=200),
            actions=[
                ft.TextButton("Cancelar", on_click=cerrar_dialogo_pago),
                ft.FilledButton("Siguiente →", bgcolor=ft.Colors.ORANGE, on_click=mostrar_qr_confirmacion)
            ]
        )
        
        page.dialog = dialogo_metodo
        dialogo_metodo.open = True
        page.update()

    def imprimir_ticket(e):
        if not carrito:
            page.snack_bar = ft.SnackBar(
                ft.Text("El carrito está vacío"),
                bgcolor=ft.Colors.RED_400
            )
            page.snack_bar.open = True
            page.update()
            return
        
        items_venta = [{
            'nombre': item['Name'],
            'cantidad': item['Cantidad'],
            'precio': item['Precio'],
            'subtotal': item['Precio'] * item['Cantidad']
        } for item in carrito]
        
        total = sum(item['subtotal'] for item in items_venta)
        
        venta_data = preparar_datos_venta(
            numero_venta=f"PREV-{datetime.now().strftime('%Y%m%d%H%M%S')}",
            items=items_venta,
            cliente_nombre="Vista Previa",
            subtotal=total,
            descuento=0,
            total=total
        )
        
        try:
            rutas = printer.imprimir_ticket(venta_data, formato='txt')
            
            page.snack_bar = ft.SnackBar(
                ft.Text(f"Ticket guardado: {rutas['txt']}"),
                bgcolor=ft.Colors.GREEN
            )
            page.snack_bar.open = True
            page.update()
            
            if os.path.exists(rutas['txt']):
                os.startfile(rutas['txt'])
        except Exception as ex:
            page.snack_bar = ft.SnackBar(
                ft.Text(f"Error: {str(ex)}"),
                bgcolor=ft.Colors.RED_400
            )
            page.snack_bar.open = True
            page.update()

    def mostrar_ayuda():
        contenido_central.controls.clear()
        
        faqs = [
            {"pregunta": "¿Cómo agregar un producto al carrito?", "respuesta": "Haz clic en el botón 'Agregar' debajo del producto. El producto se agregará al carrito en el panel derecho."},
            {"pregunta": "¿Cómo procesar una venta?", "respuesta": "Con productos en el carrito, haz clic en 'Pagar'. Selecciona método de pago, escanea el QR y confirma. El stock se descuenta automáticamente en SQL Server."},
            {"pregunta": "¿Cómo imprimir un ticket?", "respuesta": "Haz clic en 'Imprimir Voucher' para generar un ticket de vista previa. Los tickets se guardan en la carpeta 'tickets/'."},
            {"pregunta": "¿Cómo agregar nuevos productos?", "respuesta": "Ve a 'Agregar' en el menú lateral. Llena los datos con nombre, precio, stock, categoría y selecciona una imagen."},
            {"pregunta": "¿Cómo buscar un producto?", "respuesta": "Usa la barra de búsqueda en la parte superior. Los resultados se filtran automáticamente."},
            {"pregunta": "¿Qué significa el indicador de stock verde?", "respuesta": "Verde: stock disponible (>5). Naranja: stock bajo (1-5). Gris: sin stock (0)."},
            {"pregunta": "¿Cómo editar o eliminar productos?", "respuesta": "Ve a 'Inventario', usa los botones de lápiz (editar) o basura (eliminar). El historial de ventas se preserva."},
            {"pregunta": "¿El código QR es obligatorio?", "respuesta": "Sí, el código QR aparece antes de confirmar el pago. Contiene los detalles de la venta y debe ser escaneado antes de confirmar."},
        ]
        
        faq_controls = []
        for faq in faqs:
            faq_controls.append(
                ft.ExpansionTile(
                    title=ft.Text(faq["pregunta"], weight="bold", size=14),
                    subtitle=ft.Text("Haz clic para ver respuesta", size=11, color=ft.Colors.GREY_600),
                    initially_expanded=False,
                    bgcolor=ft.Colors.WHITE,
                    controls=[ft.Container(padding=15, content=ft.Text(faq["respuesta"], size=13, color=ft.Colors.GREY_700))]
                )
            )
        
        contenido_central.controls.append(
            ft.Container(
                bgcolor=ft.Colors.WHITE,
                border_radius=10,
                padding=20,
                content=ft.Column([
                    ft.Row([
                        ft.Icon(ft.Icons.HELP_OUTLINE, size=40, color=ft.Colors.ORANGE_600),
                        ft.Text("Centro de Ayuda", size=28, weight="bold")
                    ], spacing=10),
                    ft.Divider(),
                    ft.Text("Preguntas Frecuentes", size=16, weight="bold"),
                    ft.Container(height=10),
                    ft.Column(faq_controls, spacing=5),
                    ft.Container(height=20),
                    ft.Divider(),
                    ft.Container(
                        bgcolor=ft.Colors.ORANGE_50,
                        border_radius=10,
                        padding=20,
                        content=ft.Column([
                            ft.Text("¿Necesitas más ayuda?", size=18, weight="bold"),
                            ft.Container(height=10),
                            ft.Row([
                                ft.Icon(ft.Icons.PHONE, color=ft.Colors.ORANGE_600),
                                ft.Text("+51 963 499 142", size=16, weight="bold", color=ft.Colors.ORANGE_600)
                            ], spacing=10),
                            ft.Row([
                                ft.Icon(ft.Icons.EMAIL, color=ft.Colors.ORANGE_600),
                                ft.Text("luiseduardocastroguevara@gmail.com", size=14, color=ft.Colors.GREY_700)
                            ], spacing=10),
                        ])
                    )
                ], scroll=ft.ScrollMode.AUTO)
            )
        )
        page.update()

    def mostrar_agregar_productos():
        contenido_central.controls.clear()
        
        imagen_seleccionada = {"ruta": None, "ruta_completa": None}
        nombre_field = ft.TextField(label="Nombre del producto *", width=400, border_color=ft.Colors.ORANGE_400)
        precio_field = ft.TextField(label="Precio (S/) *", width=200, keyboard_type=ft.KeyboardType.NUMBER, border_color=ft.Colors.ORANGE_400)
        stock_field = ft.TextField(label="Stock inicial *", width=200, keyboard_type=ft.KeyboardType.NUMBER, border_color=ft.Colors.ORANGE_400)
        
        categorias_existentes = obtener_categorias()
        categoria_dropdown = ft.Dropdown(
            label="Categoría *",
            width=400,
            options=[ft.dropdown.Option(cat) for cat in categorias_existentes] + [ft.dropdown.Option("➕ Nueva")],
            border_color=ft.Colors.ORANGE_400
        )
        
        nueva_categoria_field = ft.TextField(label="Nueva categoría", width=400, visible=False, border_color=ft.Colors.ORANGE_400)
        
        def categoria_change(e):
            nueva_categoria_field.visible = (e.control.value == "➕ Nueva")
            page.update()
        
        categoria_dropdown.on_change = categoria_change
        
        imagen_preview = ft.Container(
            width=200,
            height=150,
            bgcolor=ft.Colors.GREY_100,
            border_radius=10,
            content=ft.Column([
                ft.Icon(ft.Icons.IMAGE, size=60, color=ft.Colors.GREY_400),
                ft.Text("Sin imagen", color=ft.Colors.GREY_500)
            ], alignment=ft.MainAxisAlignment.CENTER, horizontal_alignment=ft.CrossAxisAlignment.CENTER)
        )
        
        def seleccionar_imagen(e: ft.FilePickerResultEvent):
            if e.files and len(e.files) > 0:
                archivo = e.files[0]
                
                try:
                    if not os.path.exists("images"):
                        os.makedirs("images")
                    
                    ruta_destino = os.path.join("images", archivo.name)
                    
                    with open(archivo.path, 'rb') as f_origen:
                        with open(ruta_destino, 'wb') as f_destino:
                            f_destino.write(f_origen.read())
                    
                    imagen_seleccionada["ruta"] = f"/images/{archivo.name}"
                    imagen_seleccionada["ruta_completa"] = ruta_destino
                    
                    # Mostrar preview
                    imagen_preview.content = ft.Image(
                        src=ruta_destino,
                        width=200,
                        height=150,
                        fit=ft.ImageFit.COVER,
                        border_radius=10
                    )
                    
                    page.snack_bar = ft.SnackBar(ft.Text(f"✓ Imagen cargada: {archivo.name}"), bgcolor=ft.Colors.GREEN)
                    page.snack_bar.open = True
                    page.update()
                except Exception as ex:
                    page.snack_bar = ft.SnackBar(ft.Text(f"Error: {str(ex)}"), bgcolor=ft.Colors.RED_400)
                    page.snack_bar.open = True
                    page.update()
        
        file_picker = ft.FilePicker(on_result=seleccionar_imagen)
        page.overlay.append(file_picker)
        
        def agregar_producto_bd(e):
            if not nombre_field.value or not precio_field.value or not stock_field.value:
                page.snack_bar = ft.SnackBar(ft.Text("Completa todos los campos *"), bgcolor=ft.Colors.RED_400)
                page.snack_bar.open = True
                page.update()
                return
            
            categoria = nueva_categoria_field.value if categoria_dropdown.value == "➕ Nueva" else categoria_dropdown.value
            
            if not categoria:
                page.snack_bar = ft.SnackBar(ft.Text("Selecciona o escribe una categoría"), bgcolor=ft.Colors.RED_400)
                page.snack_bar.open = True
                page.update()
                return
            
            resultado = agregar_producto_nuevo(
                nombre=nombre_field.value,
                precio=float(precio_field.value),
                stock=int(stock_field.value),
                categoria=categoria,
                imagen=imagen_seleccionada["ruta"] or "/images/default.png"
            )
            
            if resultado["success"]:
                page.snack_bar = ft.SnackBar(ft.Text("✓ Producto agregado a SQL Server"), bgcolor=ft.Colors.GREEN)
                page.snack_bar.open = True
                
                # Limpiar campos
                nombre_field.value = ""
                precio_field.value = ""
                stock_field.value = ""
                categoria_dropdown.value = None
                nueva_categoria_field.value = ""
                nueva_categoria_field.visible = False
                imagen_seleccionada["ruta"] = None
                imagen_seleccionada["ruta_completa"] = None
                imagen_preview.content = ft.Column([
                    ft.Icon(ft.Icons.IMAGE, size=60, color=ft.Colors.GREY_400),
                    ft.Text("Sin imagen", color=ft.Colors.GREY_500)
                ], alignment=ft.MainAxisAlignment.CENTER, horizontal_alignment=ft.CrossAxisAlignment.CENTER)
                
                nonlocal productos
                productos = obtener_productos()
                render_productos()
                page.update()
            else:
                page.snack_bar = ft.SnackBar(ft.Text(f"Error: {resultado.get('error')}"), bgcolor=ft.Colors.RED_400)
                page.snack_bar.open = True
                page.update()
        
        contenido_central.controls.append(
            ft.Container(
                bgcolor=ft.Colors.WHITE,
                border_radius=10,
                padding=20,
                content=ft.Column([
                    ft.Row([
                        ft.Icon(ft.Icons.ADD_BOX, size=40, color=ft.Colors.ORANGE_600),
                        ft.Text("Agregar Producto", size=28, weight="bold")
                    ], spacing=10),
                    ft.Divider(),
                    ft.Row([
                        ft.Column([nombre_field, ft.Row([precio_field, stock_field], spacing=10), categoria_dropdown, nueva_categoria_field], spacing=15),
                        ft.Column([ft.Text("Imagen", weight="bold"), imagen_preview, ft.FilledButton("Seleccionar", icon=ft.Icons.UPLOAD_FILE, on_click=lambda _: file_picker.pick_files(allowed_extensions=["jpg", "jpeg", "png", "webp", "avif"]))], spacing=10)
                    ], spacing=40),
                    ft.Container(height=20),
                    ft.Row([
                        ft.FilledButton("Agregar Producto", icon=ft.Icons.ADD, bgcolor=ft.Colors.GREEN_600, width=200, height=45, on_click=agregar_producto_bd),
                        ft.OutlinedButton("Cancelar", width=150, height=45, on_click=lambda _: volver_inicio())
                    ], spacing=10)
                ])
            )
        )
        page.update()

    def mostrar_inventario_tabla():
        contenido_central.controls.clear()
        productos_db = obtener_productos()
        filas_tabla = []
        
        for producto in productos_db:
            stock_color = ft.Colors.GREEN_700 if producto['Stock'] > 10 else (ft.Colors.ORANGE_700 if producto['Stock'] > 0 else ft.Colors.RED_700)
            stock_bg = ft.Colors.GREEN_50 if producto['Stock'] > 10 else (ft.Colors.ORANGE_50 if producto['Stock'] > 0 else ft.Colors.RED_50)
            
            fila = ft.DataRow(cells=[
                ft.DataCell(ft.Text(str(producto['id']), size=12)),
                ft.DataCell(ft.Row([
                    ft.Image(producto['Image'], width=40, height=40, border_radius=5, fit=ft.ImageFit.COVER, error_content=ft.Icon(ft.Icons.BROKEN_IMAGE, size=30, color=ft.Colors.RED)),
                    ft.Text(producto['Name'], size=12, weight="bold")
                ], spacing=10)),
                ft.DataCell(ft.Text(producto['Categoria'], size=12)),
                ft.DataCell(ft.Text(f"S/ {producto['Precio']:.2f}", size=12, weight="bold", color=ft.Colors.ORANGE_700)),
                ft.DataCell(ft.Container(bgcolor=stock_bg, border_radius=5, padding=ft.padding.symmetric(horizontal=10, vertical=5), content=ft.Text(str(producto['Stock']), size=12, weight="bold", color=stock_color))),
                ft.DataCell(ft.Row([
                    ft.IconButton(icon=ft.Icons.EDIT, icon_color=ft.Colors.BLUE_600, tooltip="Editar", icon_size=20, on_click=lambda e, p=producto: editar_producto_dialog(p)),
                    ft.IconButton(icon=ft.Icons.DELETE, icon_color=ft.Colors.RED_600, tooltip="Eliminar", icon_size=20, on_click=lambda e, p=producto: confirmar_eliminar_producto(p))
                ], spacing=0)),
            ])
            filas_tabla.append(fila)
        
        tabla = ft.DataTable(
            bgcolor=ft.Colors.WHITE,
            border=ft.border.all(1, ft.Colors.GREY_300),
            border_radius=10,
            vertical_lines=ft.border.BorderSide(1, ft.Colors.GREY_200),
            horizontal_lines=ft.border.BorderSide(1, ft.Colors.GREY_200),
            heading_row_color=ft.Colors.ORANGE_50,
            heading_row_height=60,
            data_row_min_height=70,
            data_row_max_height=80,
            columns=[
                ft.DataColumn(ft.Text("Código", weight="bold", size=13)),
                ft.DataColumn(ft.Text("Nombre del Producto", weight="bold", size=13)),
                ft.DataColumn(ft.Text("Categoría", weight="bold", size=13)),
                ft.DataColumn(ft.Text("Precio", weight="bold", size=13)),
                ft.DataColumn(ft.Text("Stock", weight="bold", size=13)),
                ft.DataColumn(ft.Text("Acciones", weight="bold", size=13)),
                 ],
            rows=filas_tabla,
        )
        
        total_productos = len(productos_db)
        valor_inventario = sum(p['Precio'] * p['Stock'] for p in productos_db)
        stock_total = sum(p['Stock'] for p in productos_db)
        
        stats_row = ft.Row([
            ft.Container(bgcolor=ft.Colors.BLUE_50, border_radius=10, padding=15, expand=True, content=ft.Column([ft.Icon(ft.Icons.INVENTORY_2, color=ft.Colors.BLUE_700, size=30), ft.Text(str(total_productos), size=24, weight="bold", color=ft.Colors.BLUE_700), ft.Text("Total Productos", size=12, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
            ft.Container(bgcolor=ft.Colors.GREEN_50, border_radius=10, padding=15, expand=True, content=ft.Column([ft.Icon(ft.Icons.ATTACH_MONEY, color=ft.Colors.GREEN_700, size=30), ft.Text(f"S/ {valor_inventario:,.2f}", size=20, weight="bold", color=ft.Colors.GREEN_700), ft.Text("Valor Inventario", size=12, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
            ft.Container(bgcolor=ft.Colors.ORANGE_50, border_radius=10, padding=15, expand=True, content=ft.Column([ft.Icon(ft.Icons.WAREHOUSE, color=ft.Colors.ORANGE_700, size=30), ft.Text(str(stock_total), size=24, weight="bold", color=ft.Colors.ORANGE_700), ft.Text("Unidades Totales", size=12, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
        ], spacing=15)
        
        contenido_central.controls.append(
            ft.Container(bgcolor=ft.Colors.WHITE, border_radius=10, padding=20, content=ft.Column([
                ft.Row([ft.Icon(ft.Icons.INVENTORY, size=40, color=ft.Colors.ORANGE_600), ft.Text("Inventario Completo", size=28, weight="bold")], spacing=10),
                ft.Divider(),
                stats_row,
                ft.Container(height=20),
                ft.Container(content=ft.Column([tabla], scroll=ft.ScrollMode.AUTO), height=400)
            ]))
        )
        page.update()

    def editar_producto_dialog(producto):
        nombre_edit = ft.TextField(label="Nombre", value=producto['Name'], width=300)
        precio_edit = ft.TextField(label="Precio", value=str(producto['Precio']), width=150, keyboard_type=ft.KeyboardType.NUMBER)
        stock_edit = ft.TextField(label="Stock", value=str(producto['Stock']), width=150, keyboard_type=ft.KeyboardType.NUMBER)
        
        def guardar_cambios(e):
            try:
                resultado = actualizar_producto(producto['id'], nombre=nombre_edit.value, precio=float(precio_edit.value), stock=int(stock_edit.value))
                
                if resultado['success']:
                    page.snack_bar = ft.SnackBar(ft.Text("✓ Producto actualizado en SQL Server"), bgcolor=ft.Colors.GREEN)
                    page.snack_bar.open = True
                    dialog.open = False
                    
                    nonlocal productos
                    productos = obtener_productos()
                    mostrar_inventario_tabla()
                else:
                    page.snack_bar = ft.SnackBar(ft.Text(f"Error: {resultado.get('error', 'Error desconocido')}"), bgcolor=ft.Colors.RED_400)
                    page.snack_bar.open = True
            except ValueError:
                page.snack_bar = ft.SnackBar(ft.Text("Error: Precio y Stock deben ser números válidos"), bgcolor=ft.Colors.RED_400)
                page.snack_bar.open = True
            page.update()
        
        def cerrar_dialog(e):
            dialog.open = False
            page.update()
        
        dialog = ft.AlertDialog(
            title=ft.Text("✏ Editar Producto"),
            content=ft.Column([nombre_edit, ft.Row([precio_edit, stock_edit], spacing=10)], tight=True, height=150),
            actions=[ft.TextButton("Cancelar", on_click=cerrar_dialog), ft.FilledButton("Guardar", bgcolor=ft.Colors.GREEN, on_click=guardar_cambios)]
        )
        
        page.dialog = dialog
        dialog.open = True
        page.update()

    def confirmar_eliminar_producto(producto):
        def eliminar(e):
            resultado = eliminar_producto(producto['id'])
            
            if resultado['success']:
                page.snack_bar = ft.SnackBar(ft.Text("✓ Producto eliminado (historial preservado)"), bgcolor=ft.Colors.GREEN)
                page.snack_bar.open = True
                dialog.open = False
                
                nonlocal productos
                productos = obtener_productos()
                mostrar_inventario_tabla()
            else:
                page.snack_bar = ft.SnackBar(ft.Text(f"Error: {resultado.get('error', 'Error desconocido')}"), bgcolor=ft.Colors.RED_400)
                page.snack_bar.open = True
            page.update()
        
        def cerrar(e):
            dialog.open = False
            page.update()
        
        dialog = ft.AlertDialog(
            title=ft.Text("⚠ Confirmar Eliminación", color=ft.Colors.RED),
            content=ft.Column([
                ft.Text(f"¿Eliminar el producto:", weight="bold"),
                ft.Text(f"'{producto['Name']}'?", color=ft.Colors.ORANGE_700),
                ft.Container(height=10),
                ft.Container(bgcolor=ft.Colors.BLUE_50, padding=10, border_radius=8, content=ft.Column([
                    ft.Row([ft.Icon(ft.Icons.INFO, color=ft.Colors.BLUE_700, size=20), ft.Text("Importante:", weight="bold", color=ft.Colors.BLUE_700)]),
                    ft.Text("• Se desactivará pero NO se borrará", size=12),
                    ft.Text("• El historial de ventas se preserva", size=12),
                ]))
            ], tight=True, height=180),
            actions=[ft.TextButton("Cancelar", on_click=cerrar), ft.FilledButton("Sí, Eliminar", bgcolor=ft.Colors.RED, on_click=eliminar)]
        )
        
        page.dialog = dialog
        dialog.open = True
        page.update()

    def mostrar_reportes():
        contenido_central.controls.clear()
        ventas_por_dia = obtener_ventas_por_dia(30)
        stats = obtener_estadisticas_generales()
        
        stats_cards = ft.Row([
            ft.Container(bgcolor=ft.Colors.BLUE_50, border_radius=10, padding=20, expand=True, content=ft.Column([ft.Icon(ft.Icons.SHOPPING_CART, color=ft.Colors.BLUE_700, size=35), ft.Text(str(stats['total_ventas']), size=28, weight="bold", color=ft.Colors.BLUE_700), ft.Text("Total Ventas", size=13, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
            ft.Container(bgcolor=ft.Colors.GREEN_50, border_radius=10, padding=20, expand=True, content=ft.Column([ft.Icon(ft.Icons.ATTACH_MONEY, color=ft.Colors.GREEN_700, size=35), ft.Text(f"S/ {stats['suma_ventas']:,.2f}", size=24, weight="bold", color=ft.Colors.GREEN_700), ft.Text("Ingresos Totales", size=13, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
            ft.Container(bgcolor=ft.Colors.ORANGE_50, border_radius=10, padding=20, expand=True, content=ft.Column([ft.Icon(ft.Icons.TODAY, color=ft.Colors.ORANGE_700, size=35), ft.Text(f"S/ {stats['suma_ventas_hoy']:,.2f}", size=24, weight="bold", color=ft.Colors.ORANGE_700), ft.Text("Ventas Hoy", size=13, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
            ft.Container(bgcolor=ft.Colors.RED_50, border_radius=10, padding=20, expand=True, content=ft.Column([ft.Icon(ft.Icons.WARNING, color=ft.Colors.RED_700, size=35), ft.Text(str(stats['productos_stock_bajo']), size=28, weight="bold", color=ft.Colors.RED_700), ft.Text("Stock Bajo", size=13, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER)),
        ], spacing=15)
        
        if ventas_por_dia:
            max_venta = max(v['total_vendido'] for v in ventas_por_dia) if ventas_por_dia else 1
            barras = []
            
            for venta in ventas_por_dia[-15:]:
                altura = (venta['total_vendido'] / max_venta) * 200 if max_venta > 0 else 0
                try:
                    # Manejar fechas de SQL Server
                    if isinstance(venta['dia'], str):
                        fecha_obj = datetime.strptime(venta['dia'], '%Y-%m-%d')
                    else:
                        fecha_obj = venta['dia']
                    dia_texto = fecha_obj.strftime('%d/%m')
                except:
                    dia_texto = str(venta['dia'])[-5:]
                
                barras.append(ft.Container(content=ft.Column([
                    ft.Container(width=35, height=altura, bgcolor=ft.Colors.ORANGE_400, border_radius=ft.border_radius.only(top_left=5, top_right=5), tooltip=f"{dia_texto}\nS/ {venta['total_vendido']:.2f}\n{venta['cantidad_ventas']} ventas"),
                    ft.Text(dia_texto, size=9, color=ft.Colors.GREY_600, text_align=ft.TextAlign.CENTER),
                    ft.Text(f"S/{venta['total_vendido']:.0f}", size=8, color=ft.Colors.ORANGE_700, weight="bold", text_align=ft.TextAlign.CENTER),
                ], horizontal_alignment=ft.CrossAxisAlignment.CENTER, spacing=2), expand=True))
            
            grafico = ft.Container(bgcolor=ft.Colors.WHITE, border_radius=10, padding=20, content=ft.Column([
                ft.Text("📊 Ventas por Día (Últimos 15 días)", size=18, weight="bold"),
                ft.Divider(),
                ft.Container(content=ft.Row(barras, alignment=ft.MainAxisAlignment.SPACE_AROUND, vertical_alignment=ft.CrossAxisAlignment.END), height=280, padding=ft.padding.only(bottom=40)),
                ft.Row([ft.Icon(ft.Icons.CALENDAR_TODAY, size=16, color=ft.Colors.GREY_600), ft.Text("Día", size=12, color=ft.Colors.GREY_600), ft.Container(width=20), ft.Icon(ft.Icons.ATTACH_MONEY, size=16, color=ft.Colors.ORANGE_600), ft.Text("Monto en Soles", size=12, color=ft.Colors.GREY_600)], alignment=ft.MainAxisAlignment.CENTER)
            ]))
        else:
            grafico = ft.Container(bgcolor=ft.Colors.GREY_100, border_radius=10, padding=40, content=ft.Column([ft.Icon(ft.Icons.SHOW_CHART, size=60, color=ft.Colors.GREY_400), ft.Text("No hay ventas registradas", size=16, color=ft.Colors.GREY_600)], horizontal_alignment=ft.CrossAxisAlignment.CENTER))
        
        contenido_central.controls.append(
            ft.Container(content=ft.Column([
                ft.Row([ft.Icon(ft.Icons.BAR_CHART, size=40, color=ft.Colors.ORANGE_600), ft.Text("Reportes y Estadísticas", size=28, weight="bold")], spacing=10),
                ft.Divider(),
                stats_cards,
                ft.Container(height=20),
                grafico,
            ], scroll=ft.ScrollMode.AUTO))
        )
        page.update()

    def mostrar_configuracion():
        contenido_central.controls.clear()
        tema_switch = ft.Switch(label="Modo Oscuro", value=page.theme_mode == ft.ThemeMode.DARK, active_color=ft.Colors.ORANGE_600)
        
        def cambiar_tema(e):
            if tema_switch.value:
                page.theme_mode = ft.ThemeMode.DARK
                page.bgcolor = ft.Colors.GREY_900
            else:
                page.theme_mode = ft.ThemeMode.LIGHT
                page.bgcolor = ft.Colors.INDIGO_100
            page.snack_bar = ft.SnackBar(ft.Text(f"Tema {'oscuro' if tema_switch.value else 'claro'} activado"), bgcolor=ft.Colors.GREEN)
            page.snack_bar.open = True
            page.update()
        
        tema_switch.on_change = cambiar_tema
        
        contenido_central.controls.append(
            ft.Container(content=ft.Column([
                ft.Row([ft.Icon(ft.Icons.SETTINGS, size=40, color=ft.Colors.ORANGE_600), ft.Text("Configuración", size=28, weight="bold")], spacing=10),
                ft.Divider(),
                ft.Container(bgcolor=ft.Colors.WHITE, border_radius=10, padding=20, content=ft.Column([
                    ft.Row([ft.Icon(ft.Icons.PALETTE, size=30, color=ft.Colors.ORANGE_600), ft.Text("Apariencia", size=20, weight="bold")], spacing=10),
                    ft.Divider(),
                    tema_switch,
                ])),
                ft.Container(height=15),
                ft.Container(bgcolor=ft.Colors.ORANGE_50, border_radius=10, padding=20, content=ft.Column([
                    ft.Row([ft.Icon(ft.Icons.INFO, size=30, color=ft.Colors.ORANGE_600), ft.Text("Información del Sistema", size=20, weight="bold")], spacing=10),
                    ft.Divider(),
                    ft.Row([ft.Text("Base de Datos:", weight="bold"), ft.Text("SQL Server", color=ft.Colors.GREEN_700)], spacing=10),
                    ft.Row([ft.Text("Desarrollador:", weight="bold"), ft.Text("Eduardo Castro", color=ft.Colors.GREY_700)], spacing=10),
                ]))
            ], scroll=ft.ScrollMode.AUTO))
        )
        page.update()

    def cambiar_seccion(e):
        """MENÚ CON BOTÓN INICIO (SIN Equipos/Componentes/Periféricos)"""
        seccion = e.control.selected_index
        contenido_central.controls.clear()
        
        if seccion == 0:  # INICIO
            volver_inicio()
        elif seccion == 1:  # Inventario
            mostrar_inventario_tabla()
        elif seccion == 2:  # Reportes
            mostrar_reportes()
        elif seccion == 3:  # Agregar
            mostrar_agregar_productos()
        elif seccion == 4:  # Configuración
            mostrar_configuracion()
        elif seccion == 5:  # Ayuda
            mostrar_ayuda()
        
        page.update()

    # SIDEBAR CON BOTÓN INICIO
    sidebar = ft.Container(
        bgcolor=ft.Colors.WHITE,
        col=1,
        border_radius=10,
        content=ft.NavigationRail(
            destinations=[
                ft.NavigationRailDestination(icon=ft.Icons.HOME, label="Inicio"),
                ft.NavigationRailDestination(icon=ft.Icons.INVENTORY, label="Inventario"),
                ft.NavigationRailDestination(icon=ft.Icons.BAR_CHART, label="Reportes"),
                ft.NavigationRailDestination(icon=ft.Icons.ADD_BOX, label="Agregar"),
                ft.NavigationRailDestination(icon=ft.Icons.SETTINGS, label="Configuración"),
                ft.NavigationRailDestination(icon=ft.Icons.HELP, label="Ayuda"),
            ],
            expand=True,
            on_change=cambiar_seccion,
        )
    )

    page.appbar = ft.AppBar(
        leading=ft.Container(
            content=ft.Row([
                ft.Text('Inventario', weight='bold', size=25),
                ft.Text('Local', weight='bold', color=ft.Colors.ORANGE_500, size=25)
            ]),
            padding=ft.padding.only(left=14, top=8)
        ),
        actions=[
            ft.Row([
                ft.TextButton('Inicio', on_click=volver_inicio),
                ft.TextButton('Clientes'),
                ft.TextButton('Cajero'),
                ft.FilledButton('Nuevas órdenes', bgcolor=ft.Colors.ORANGE),
                ft.IconButton(icon=ft.Icons.NOTIFICATIONS)
            ])
        ]
    )

    side_right = ft.Container(
        padding=10,
        bgcolor=ft.Colors.WHITE,
        col=2.5,
        border_radius=10,
        content=ft.Column([
            ft.Container(
                height=200,
                content=ft.Column([
                    ft.Text("Pedido nro: 145600", color=ft.Colors.GREY_500),
                    ft.Divider(height=1),
                    ft.Row([
                        ft.Icon(ft.Icons.SHOPPING_CART, size=40, color=ft.Colors.ORANGE),
                        ft.Column([
                            ft.Text("Inventario Local", weight="bold", size=22),
                            ft.Text("luiseduardocastroguevara@gmail.com", color=ft.Colors.GREY_400, size=11)
                        ])
                    ]),
                    ft.Row([
                        ft.Text("Mesa 04", weight="bold", color=ft.Colors.ORANGE),
                        ft.Text("Orden #20", weight="bold")
                    ], alignment=ft.MainAxisAlignment.SPACE_BETWEEN),
                    ft.Divider(height=1),
                ])
            ),
            list_view_pedido,
            ft.Container(
                padding=10,
                height=130,
                content=ft.Column([
                    ft.Divider(height=1),
                    ft.Row([
                        ft.Column([
                            ft.Text("Total", weight="bold"),
                            items_count_text
                        ]),
                        total_text
                    ], alignment=ft.MainAxisAlignment.SPACE_BETWEEN)
                ])
            ),
            ft.CupertinoButton("Imprimir Voucher", bgcolor=ft.Colors.INDIGO_50, color=ft.Colors.GREY_700, width=300, on_click=imprimir_ticket),
            ft.CupertinoButton("Pagar", bgcolor=ft.Colors.GREEN_600, width=300, on_click=procesar_pago)
        ])
    )

    body = ft.Container(col=8.5, content=contenido_central)
    main_container = ft.Container(padding=8, expand=True, bgcolor=ft.Colors.INDIGO_100, content=ft.ResponsiveRow([sidebar, body, side_right]))

    page.add(main_container)
    volver_inicio()
