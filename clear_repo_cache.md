git clean -fdx
pip install -e .
jlpm install
jlpm run build
jlpm run build:core
jupyter lab build