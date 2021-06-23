# Base off of Python 3 image
FROM python:3

# Do not prompt for input
ENV PYTHONUNBUFFERED=1

# Make a new directory, in root, called "app", which is where commands will run by default
WORKDIR /app

# Install our dependencies ( it is better to do this with requirements.txt, but skipping that for the Demo)
RUN pip install Django

# Copy our source code over
COPY . .

# EXPOSE doesn't do anything, except inform other people/users that the given port is use by the container.
EXPOSE 8000
CMD [ "python", "manage.py", "runserver", "0.0.0.0:8000" ]
