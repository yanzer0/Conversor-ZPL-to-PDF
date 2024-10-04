# Conversor-ZPL-to-PDF

# Importando bibliotecas
    import os
    import requests
    from PyPDF2 import PdfReader, PdfWriter
    from reportlab.lib.pagesizes import letter
    from reportlab.pdfgen import canvas
    from io import BytesIO
    from PIL import Image
    from tkinter import Tk, Label, Button, filedialog
    from tkinter import messagebox

# Converte um arquivo ZPL para PDF usando a API Labelary
    def zpl_to_pdf(zpl_file, pdf_file):
        with open(zpl_file, 'r') as file:
            zpl_content = file.read()

  # Enviar o conteúdo ZPL para a API Labelary
    url = 'http://api.labelary.com/v1/printers/8dpmm/labels/4x6/0/'
    headers = {'Accept': 'image/png'}
    response = requests.post(url, headers=headers, data=zpl_content)

    if response.status_code == 200:
  # Salvar a imagem recebida como PNG
        label_image = Image.open(BytesIO(response.content))

  # Obter dimensões da imagem original
        img_width, img_height = label_image.size

  # Calcular a nova altura e largura mantendo a proporção, ajustando para o tamanho da página PDF
        max_width, max_height = letter  # Tamanho da página 'letter'
        aspect_ratio = img_width / img_height

  # Ajustar a imagem mantendo as proporções
        if aspect_ratio > 1:  # Se a imagem for mais larga que alta
            new_width = max_width - 20  # Definir largura com margem
            new_height = new_width / aspect_ratio
        else:  # Se a imagem for mais alta que larga
            new_height = max_height - 20  # Definir altura com margem
            new_width = new_height * aspect_ratio

  # Criar o PDF usando a imagem redimensionada
        c = canvas.Canvas(pdf_file, pagesize=letter)
        temp_image_path = "temp_image.png"
        label_image.save(temp_image_path)
        
  # Centralizar a imagem no PDF
        x_margin = (letter[0] - new_width) / 2
        y_margin = (letter[1] - new_height) / 2

  # Desenhar a imagem no PDF na nova escala e centralizada
        c.drawImage(temp_image_path, x_margin, y_margin, width=new_width, height=new_height)
        c.showPage()
        c.save()

  # Remover o arquivo de imagem temporário
        os.remove(temp_image_path)
    else:
        print(f"Erro na conversão do arquivo ZPL. Status code: {response.status_code}")

# Mescla dois PDFs, ajustando o tamanho das páginas para que sejam iguais.
    def mesclar_pdf(pdf_path1, pdf_path2, output_pdf):
        pdf_writer = PdfWriter()
        pdf_reader1 = PdfReader(pdf_path1)
        pdf_reader2 = PdfReader(pdf_path2)

# Mesclar a primeira página do primeiro PDF
    for page in pdf_reader1.pages:
        page.scale_to(letter[0], letter[1])  # Garantir que a página 1 tenha o tamanho de uma folha A4
        pdf_writer.add_page(page)

# Mesclar a primeira página do segundo PDF
    for page in pdf_reader2.pages:
        page.scale_to(letter[0], letter[1])  # Garantir que a página 2 também tenha o tamanho de uma folha A4
        pdf_writer.add_page(page)

# Salvar o arquivo final mesclado
    with open(output_pdf, 'wb') as output_file:
        pdf_writer.write(output_file)

# Função para selecionar pastas com uma mensagem informando o tipo de pasta.
    def selecionar_pasta(tipo_pasta):
        messagebox.showinfo("Seleção de Pasta", f"Por favor, selecione a pasta de {tipo_pasta}.")
        pasta = filedialog.askdirectory()
        return pasta

    def processar():
        pasta_etiquetas = selecionar_pasta("etiquetas (arquivos ZPL)")
        pasta_saida_pdf = selecionar_pasta("saída para os PDFs convertidos")
        pasta_declaracoes = selecionar_pasta("declarações de conteúdo (arquivos PDF)")
        pasta_saida_mesa = selecionar_pasta("saída para os arquivos mesclados")

    if not pasta_etiquetas or not pasta_saida_pdf or not pasta_declaracoes or not pasta_saida_mesa:
        messagebox.showwarning("Aviso", "Por favor, selecione todas as pastas.")
        return

  # Converter arquivos ZPL para PDF
    zpl_files = [f for f in os.listdir(pasta_etiquetas) if f.endswith('.zpl')]
    
    for zpl_file in zpl_files:
        zpl_path = os.path.join(pasta_etiquetas, zpl_file)
        pdf_file_name = f"{os.path.splitext(zpl_file)[0]}.pdf"
        pdf_path = os.path.join(pasta_saida_pdf, pdf_file_name)
        
        zpl_to_pdf(zpl_path, pdf_path)

  # Listar arquivos PDF na pasta de declarações
    declaracoes = [f for f in os.listdir(pasta_declaracoes) if f.endswith('.pdf')]
    
  # Mesclar os PDFs convertidos com as declarações
    for zpl_file in zpl_files:
        nome_base = os.path.splitext(zpl_file)[0]  # Nome base do arquivo ZPL convertido
        pdf_path = os.path.join(pasta_saida_pdf, f"{nome_base}.pdf")
        
        found = False
        for decl in declaracoes:
            if decl.startswith(nome_base):
                declaracao_path = os.path.join(pasta_declaracoes, decl)
                found = True
                break

        if found:
            arquivo_saida = os.path.join(pasta_saida_mesa, f"{nome_base} (mesclado).pdf")
            mesclar_pdf(pdf_path, declaracao_path, arquivo_saida)

    messagebox.showinfo("Sucesso", "Processamento concluído!")

# Criando a interface gráfica
    root = Tk()
    root.title("Conversor ZPL para PDF e Mesclagem")

    label = Label(root, text="Clique no botão para selecionar as pastas e iniciar o processo")
    label.pack(pady=10)

    botao = Button(root, text="Selecionar Pastas e Processar", command=processar)
    botao.pack(pady=20)

    root.mainloop()
