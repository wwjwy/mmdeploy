## How to support new backends

MMDeploy supports a number of backend engines. We welcome the contribution of new backends. In this tutorial, we will introduce the general procedures to support a new backend in MMDeploy.

### Prerequisites

Before contributing the codes, there are some requirements for the new backend that need to be checked:

* The backend must support ONNX as IR.
* If the backend requires model files or weight files other than a ".onnx" file, a conversion tool that converts the ".onnx" file to model files and weight files is required. The tool can be a Python API, a script, or an executable program.
* It is highly recommended that the backend provides a Python interface to load the backend files and inference for validation.

### Support backend conversion

The backends in MMDeploy must support the ONNX. The backend loads the ".onnx" file directly, or converts the ".onnx" to its own format using the conversion tool. In this section, we will introduce the steps to support backend conversion.

1. Add backend constant in `mmdeploy/utils/constants.py` that denotes the name of the backend.

    **Example**:

    ```Python
    # mmdeploy/utils/constants.py

    class Backend(AdvancedEnum):
        # Take TensorRT as an example
        TENSORRT = 'tensorrt'
    ```

2. Add a corresponding package (a folder with `__init__.py`) in `mmdeploy/backend/`. For example, `mmdeploy/backend/tensorrt`. In the `__init__.py`, there must be a function named `is_available` which checks if users have installed the backend library. If the check is passed, then the remaining files of the package will be loaded.

    **Example**:

    ```Python
    # mmdeploy/backend/tensorrt/__init__.py

    def is_available():
        return importlib.util.find_spec('tensorrt') is not None


    if is_available():
        from .utils import create_trt_engine, load_trt_engine, save_trt_engine
        from .wrapper import TRTWrapper

        __all__ = [
            'create_trt_engine', 'save_trt_engine', 'load_trt_engine', 'TRTWrapper'
        ]
    ```

3. Create a config file in `configs/_base_/backends` (e.g., `configs/_base_/backends/tensorrt.py`).  If the backend just takes the '.onnx' file as input, the new config can be simple. The config of the backend only consists of one field denoting the name of the backend (which should be same as the name in `mmdeploy/utils/constants.py`).

    **Example**:

    ```python
    backend_config = dict(type='onnxruntime')
    ```

    If the backend requires other files, then the arguments for the conversion from ".onnx" file to backend files should be included in the config file.

    **Example:**

    ```Python

    backend_config = dict(
        type='tensorrt',
        common_config=dict(
            fp16_mode=False, max_workspace_size=0))
    ```

    After possessing a base backend config file, you can easily construct a complete deploy config through inheritance. Please refer to our [config tutorial](how_to_write_config.md) for more details. Here is an example:

    ```Python
    _base_ = ['../_base_/backends/onnxruntime.py']

    codebase_config = dict(type='mmcls', task='Classification')
    onnx_config = dict(input_shape=None)
    ```

4. If the backend requires model files or weight files other than a ".onnx" file, create a `onnx2backend.py` file in the corresponding folder (e.g., create `mmdeploy/backend/tensorrt/onnx2tensorrt.py`). Then add a conversion function `onnx2backend` in the file. The function should convert a given ".onnx" file to the required backend files in a given work directory. There are no requirements on other parameters of the function and the implementation details. You can use any tools for conversion. Here are some examples:

    **Use Python script:**

    ```Python
    def onnx2openvino(input_info: Dict[str, Union[List[int], torch.Size]],
                      output_names: List[str], onnx_path: str, work_dir: str):

        input_names = ','.join(input_info.keys())
        input_shapes = ','.join(str(list(elem)) for elem in input_info.values())
        output = ','.join(output_names)

        mo_args = f'--input_model="{onnx_path}" '\
                  f'--output_dir="{work_dir}" ' \
                  f'--output="{output}" ' \
                  f'--input="{input_names}" ' \
                  f'--input_shape="{input_shapes}" ' \
                  f'--disable_fusing '
        command = f'mo.py {mo_args}'
        mo_output = run(command, capture_output=True, shell=True, check=True)
    ```

    **Use executable program:**

    ```Python
    def onnx2ncnn(onnx_path: str, work_dir: str):
        onnx2ncnn_path = get_onnx2ncnn_path()
        save_param, save_bin = get_output_model_file(onnx_path, work_dir)
        call([onnx2ncnn_path, onnx_path, save_param, save_bin])\
    ```

5. Define APIs in a new package in  `mmdeploy/apis`.

    **Example:**

    ```Python
    # mmdeploy/apis/ncnn/__init__.py

    from mmdeploy.backend.ncnn import is_available

    __all__ = ['is_available']

    if is_available():
        from mmdeploy.backend.ncnn.onnx2ncnn import (onnx2ncnn,
                                                     get_output_model_file)
        __all__ += ['onnx2ncnn', 'get_output_model_file']
    ```

    Then add the codes about conversion to `tools/deploy.py` using these APIs if necessary.

    **Example:**

    ```Python
    # tools/deploy.py
    # ...
        elif backend == Backend.NCNN:
            from mmdeploy.apis.ncnn import is_available as is_available_ncnn

            if not is_available_ncnn():
                logging.error('ncnn support is not available.')
                exit(-1)

            from mmdeploy.apis.ncnn import onnx2ncnn, get_output_model_file

            backend_files = []
            for onnx_path in onnx_files:
                create_process(
                    f'onnx2ncnn with {onnx_path}',
                    target=onnx2ncnn,
                    args=(onnx_path, args.work_dir),
                    kwargs=dict(),
                    ret_value=ret_value)
                backend_files += get_output_model_file(onnx_path, args.work_dir)
    # ...
    ```

6. Convert the models of OpenMMLab to backends (if necessary) and inference on backend engine. If you find some incompatible operators when testing, you can try to rewrite the original model for the backend following the [rewriter tutorial](how_to_support_new_models.md) or add custom operators.

7. Add docstring and unit tests for new code :).

### Support backend inference

Although the backend engines are usually implemented in C/C++, it is convenient for testing and debugging if the backend provides Python inference interface. We encourage the contributors to support backend inference in the Python interface of MMDeploy. In this section we will introduce the steps to support backend inference.

1. Add a file named `wrapper.py` to corresponding folder in `mmdeploy/backend/{backend}`. For example, `mmdeploy/backend/tensorrt/wrapper.py`. This module should implement and register a wrapper class that inherits the base class `BaseWrapper` in `mmdeploy/backend/base/base_wrapper.py`.

    **Example:**

    ```Python
    from mmdeploy.utils import Backend
    from ..base import BACKEND_WRAPPER, BaseWrapper

    @BACKEND_WRAPPER.register_module(Backend.TENSORRT.value)
    class TRTWrapper(BaseWrapper):
    ```

2. The wrapper class can initialize the engine in `__init__` function and inference in `forward` function. Note that the `__init__` function must take a parameter `output_names` and pass it to base class to determine the orders of output tensors. The input and output variables of `forward` should be dictionaries denoting the name and value of the tensors.

3. For the convenience of performance testing, the class should define a "execute" function that only calls the inference interface of the backend engine. The `forward` function should call the "execute" function after preprocessing the data.

    **Example:**

    ```Python
    from mmdeploy.utils import Backend
    from mmdeploy.utils.timer import TimeCounter
    from ..base import BACKEND_WRAPPER, BaseWrapper

    @BACKEND_WRAPPER.register_module(Backend.ONNXRUNTIME.value)
    class ORTWrapper(BaseWrapper):

        def __init__(self,
                     onnx_file: str,
                     device: str,
                     output_names: Optional[Sequence[str]] = None):
            # Initialization
            # ...
            super().__init__(output_names)

        def forward(self, inputs: Dict[str,
                                       torch.Tensor]) -> Dict[str, torch.Tensor]:
            # Fetch data
            # ...

            self.__ort_execute(self.io_binding)

    		# Postprocess data
            # ...

        @TimeCounter.count_time()
        def __ort_execute(self, io_binding: ort.IOBinding):
    		# Only do the inference
            self.sess.run_with_iobinding(io_binding)
    ```

4. Add a default initialization method for the new wrapper in `mmdeploy/codebase/base/backend_model.py`

    **Example:**

    ```Python
        @staticmethod
        def _build_wrapper(backend: Backend,
                           backend_files: Sequence[str],
                           device: str,
                           output_names: Optional[Sequence[str]] = None):
            if backend == Backend.ONNXRUNTIME:
                from mmdeploy.backend.onnxruntime import ORTWrapper
                return ORTWrapper(
                    onnx_file=backend_files[0],
                    device=device,
                    output_names=output_names)
    ```

5. Add docstring and unit tests for new code :).
