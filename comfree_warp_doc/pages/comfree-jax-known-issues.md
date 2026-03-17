# ComFree Jax Known Issues

This page tracks current known issues and limitations of the ComFree Jax implementation.

- **JAX version compatibility**: `mjx` may require specific JAX/JAXLIB versions. If you hit build or runtime errors, try aligning versions with the `requirements.txt` used by the project.
- **GPU memory use**: Large models or large batch sizes can consume significant GPU memory. Reduce batch size or run on CPU if you hit out-of-memory errors.
- **MuJoCo API changes**: If you upgrade MuJoCo, the JAX bindings may need updates. Check for breaking changes in MuJoCo’s XML schema or API.

If you discover a new issue, please open an issue in the repository and include a minimal reproduction.
