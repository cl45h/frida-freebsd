# Frida en FreeBSD

Guia para compilar [Frida](https://frida.re) desde source en FreeBSD 15.0.

Frida dice soportar FreeBSD pero no hay binarios precompilados ni documentacion para compilarlo. Esta guia documenta los parches necesarios y los pasos para lograr un build funcional.

Probado en FreeBSD 15.0-RELEASE con i9-13900KF.

## Que funciona

- `frida-ps` - listar procesos
- `frida-trace` - tracear funciones
- `frida` - CLI interactivo
- Python bindings (`import frida`)

## Dependencias

```sh
pkg install vala bison py311-pip py311-setuptools npm node uthash
```

## Parches necesarios

Se necesitan dos parches para que compile:

### 1. `compat/build.py` (frida-core)

Cuando se compila con `--without-prebuilds`, el set `allowed_prebuilds` queda como `{''}` (set con string vacio) en vez de `set()`. Esto hace que el build intente descargar prebuilds que no existen para FreeBSD y falle.

El parche esta en `0001-fix-empty-prebuilds-set-for-freebsd.patch`.

### 2. `config.h` (generado)

Al compilar sin SDK precompilado, no se genera el define `FRIDA_LIBDIR_NAME`. Hay que agregarlo manualmente despues del configure:

```c
#define FRIDA_LIBDIR_NAME "frida-17.0"
```

## Build paso a paso

### 1. Compilar el fork de Vala de Frida

Frida usa su propio fork de Vala que no esta en los repos de FreeBSD:

```sh
git clone https://github.com/frida/vala
cd vala
git checkout v0.58.0-frida
./autogen.sh
make -j$(sysctl -n hw.ncpu)
sudo make install
ldconfig -m /usr/local/lib/vala-0.58
```

### 2. Clonar Frida

```sh
git clone --recurse-submodules https://github.com/frida/frida
cd frida
```

### 3. Aplicar el parche

```sh
cd subprojects/frida-core
patch -p1 < /path/to/0001-fix-empty-prebuilds-set-for-freebsd.patch
cd ../..
```

### 4. Configure

```sh
PYTHON=python3.11 FRIDA_ALLOWED_PREBUILDS="" ./configure --without-prebuilds=toolchain,sdk --disable-frida-tools
```

### 5. Agregar define faltante

```sh
echo '#define FRIDA_LIBDIR_NAME "frida-17.0"' >> build/subprojects/frida-core/config.h
```

### 6. Build

```sh
FRIDA_ALLOWED_PREBUILDS="" ninja -C build -j$(sysctl -n hw.ncpu)
```

### 7. Instalar Python bindings

```sh
cp build/subprojects/frida-python/frida/_frida/_frida.abi3.so \
   $(python3.11 -c "import site; print(site.getsitepackages()[0])")/frida/_frida.abi3.so

pip-3.11 install frida-tools --no-deps
```

### 8. Verificar

```sh
frida-ps
frida --version
python3.11 -c "import frida; print(frida.enumerate_devices())"
```

## Contribuir upstream

Estos parches son candidatos para un PR al repo oficial de Frida:

- **`frida/frida-core`**: El fix de `compat/build.py` es un bug real - el manejo de `allowed_prebuilds` como set vacio falla independientemente de la plataforma.
- **`frida/frida`**: Agregar FreeBSD al CI o al menos documentar el build.

Issues/PRs van a: https://github.com/frida/frida/issues
